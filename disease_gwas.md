# Disease report / GWAS

## Parameters

* `medgen_cid` MedGen CID
  * default: C0023467 
  * example: C0023467(acute myeloid leukemia), C2675520(breast-ovarian cancer)  
  
## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}


## `medgen2efo`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX efo: <http://www.ebi.ac.uk/efo/>
PREFIX medgen: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX mesh: <http://id.mesh.nlm.gov>
PREFIX mo: <http://med2rdf/ontology/medgen#>
PREFIX mondo: <http://purl.obolibrary.org/obo/mondo#>
PREFIX oboinowl: <http://www.geneontology.org/formats/oboInOwl#>

SELECT DISTINCT ?efo
FROM <http://togovar.biosciencedbc.jp/medgen>
FROM <http://togovar.biosciencedbc.jp/mondo>
WHERE {
  VALUES ?medgen { medgen:{{ medgen_cid }}  }
  
  GRAPH <http://togovar.biosciencedbc.jp/medgen> {
    ?medgen a mo:ConceptID.
    ?medgen rdfs:label ?medgen_label.
    ?medgen mo:mgconso ?mgconso.
    ?mgconso dct:source mo:MONDO.
    ?mgconso rdfs:seeAlso ?mondo.
  }

  GRAPH <http://togovar.biosciencedbc.jp/mondo> {
    ?mondo oboinowl:hasDbXref ?dbxref.
    FILTER(STRSTARTS(STR(?dbxref), "EFO:")).
    BIND(URI(CONCAT("http://www.ebi.ac.uk/efo/EFO_", SUBSTR(?dbxref,5))) AS ?efo).
  }
}
```

## `efo`
```javascript
({medgen2efo}) => {
  return medgen2efo.results.bindings.map(b => b.efo.value);
}
```

## `efo2gwas`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX efo: <http://www.ebi.ac.uk/efo/>
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX mondo: <http://purl.obolibrary.org/obo/mondo#>
PREFIX oboinowl: <http://www.geneontology.org/formats/oboInOwl#>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT DISTINCT ?assoc
       ?variant_and_risk_allele
       ?raf
       ?p_value
       ?odds_ratio
       ?ci_text
       ?beta
       ?beta_unit
       ?mapped_trait
       ?mapped_trait_uri
       ?pubmed_id
       ?pubmed as ?pubmed_uri
       ?description as ?study_detail
       ?study
       ?initial_sample_size
       ?replication_sample_size
FROM <http://togovar.biosciencedbc.jp/efo>
FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
WHERE {
 VALUES ?conditions { <{{ efo }}> }

 GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
    ?assoc a gwas:Association ;
      terms:mapped_trait_uri ?conditions ;
      terms:reported_genes ?reported_genes ;
      terms:strongest_snp_risk_allele ?variant_and_risk_allele ;
      terms:risk_allele_frequency ?raf ;
      terms:snps ?rs_id ;
      terms:p_value ?p_value ;
      terms:odds_ratio ?odds_ratio ;
      terms:beta ?beta ;
      terms:beta_unit ?beta_unit ;
      terms:ci_text ?ci_text ;
      terms:study ?study ;
      dct:date ?association_date ;
      dct:references ?pubmed ;
      gwas:has_pubmed_id ?pubmed_id .

    OPTIONAL{
       ?assoc terms:mapped_trait_uri ?mapped_trait_uri .
    }
    ?study dct:identifier ?study_id ;
      dct:description ?description ;
      terms:initial_sample_size ?initial_sample_size ;
      terms:replication_sample_size ?replication_sample_size .
 }
 GRAPH <http://togovar.biosciencedbc.jp/efo>{
   ?mapped_trait_uri rdfs:label ?mapped_trait.
 }
}
```

## `rs2tgv`
```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, efo2gwas}) => {
  let rs_ids = [];
  const num_of_rs_ids_per_fetch = 200;
  let rs_ids_per_fetch = [];
  let rs2tgv = {};
  
  const api_options = {
    method: 'GET',
    headers: {
      'Accept': 'application/json',
    },
  };

  efo2gwas.results.bindings.map(binding => {
    return binding.variant_and_risk_allele.value.split("; ").forEach(rs => {
      rs_ids.push(rs.split('-')[0]);
    });
  });
  rs_ids = Array.from(new Set(rs_ids));

  while(rs_ids.length > 1){
    const api = SPARQLIST_TOGOVAR_SPARQLIST + "/api/rs2tgvid?rs_id=" + rs_ids.splice(1, num_of_rs_ids_per_fetch).join(",");  
    try{
      const json = await fetch(api, api_options).then(res => res.json()).then(json => {
        for(const rs_id of Object.keys(json)){
          rs2tgv[rs_id] = json[rs_id];
        } 
      });
    }catch(e){
      console.log("Error in rs2tgv:" + e);
    }
  }

  return rs2tgv;
}
```

## `trait_html`
```javascript
({efo2gwas})=>{
  const traits = {};
  efo2gwas.results.bindings.map(x=>{
    if (! traits[x.assoc.value]){ traits[x.assoc.value] = [] } 
    traits[x.assoc.value].push('<a href="' + x.mapped_trait_uri.value + '">' + x.mapped_trait.value + '</a>');
  })
  return traits;
}
```

## `result`

```javascript
({efo2gwas,rs2tgv,trait_html}) => {
  const res = [];

  efo2gwas.results.bindings.forEach(d => {
    const variant_and_risk_allele = d.variant_and_risk_allele.value.split(/;\s+/).map(rs_allele => {
      const rs_id = rs_allele.split('-')[0];
      const rs_risk_allele = rs_allele.split('-')[1];
      const tgv_id = rs2tgv[rs_id];
      const link_to_gwas = "<a href='https://www.ebi.ac.uk/gwas/variants/" + rs_id + "'>" + rs_id + "-" + rs_risk_allele + "</a>";
      const link_to_tgv = "<a href='/variant/" + rs2tgv[rs_id] + "'>" + rs2tgv[rs_id] + "</a>";

      return tgv_id ? "<span style='display: inline-block;'>" + link_to_gwas + "</span>(" + link_to_tgv + ")" : link_to_gwas;
    }).join('<br>');

    res.push({
        variant_and_risk_allele: variant_and_risk_allele,
        raf: d.raf.value,
        p_value: d.p_value.value,
        odds_ratio: d.odds_ratio.value,
        ci_text: d.ci_text.value,
        beta: d.beta.value,
        beta_unit: d.beta_unit.value,
        mapped_trait: trait_html[d.assoc.value].join(),
        pubmed_id: d.pubmed_id.value,
        pubmed_uri: d.pubmed_uri.value,
        study_detail: d.study.value.replace("http://www.ebi.ac.uk/gwas/studies/",""),
        study: d.study.value,
        initial_sample_size: d.initial_sample_size.value,
        replication_sample_size: d.replication_sample_size.value
    });
  });
  return res;
}
```
