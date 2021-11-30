# Disease report / GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `medgen_cid` MedGen CID
  * default: C0023467 
  * example: C0023467(acute myeloid leukemia), C2675520(breast-ovarian cancer)
* `base_url` TogoVar URL
  * default: https://togovar.biosciencedbc.jp
  
  
## Endpoint

{{ ep }}


## `medgen2efo`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX efo: <http://www.ebi.ac.uk/efo/>
PREFIX medgen: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX mesh: <http://id.mesh.nlm.gov>
PREFIX mo: <http://med2rdf/ontology/medgen#>
PREFIX mondo: <http://purl.obolibrary.org/obo/mondo#>

SELECT DISTINCT ?efo
FROM <http://togovar.biosciencedbc.jp/medgen>
FROM <http://togovar.biosciencedbc.jp/efo>
WHERE {
  VALUES ?medgen { medgen:{{ medgen_cid }} }
  
  GRAPH <http://togovar.biosciencedbc.jp/medgen> {
    ?medgen a mo:ConceptID.
    ?medgen rdfs:label ?medgen_label.
    ?medgen mo:mgconso ?mgconso.
    ?mgconso dct:source mo:MSH.
    ?mgconso rdfs:seeAlso ?mesh.
  }

   BIND(IRI(REPLACE(STR(?mesh), "http://id.nlm.nih.gov/mesh/","http://identifiers.org/mesh/")) AS ?mesh_idt)

  GRAPH <http://togovar.biosciencedbc.jp/efo>{
    ?efo mondo:exactMatch ?mesh_idt.
  }
}
```

## `efo`
```javascript
({medgen2efo}) => {
// const prefix = "http://identifiers.org/dbsnp/";
// return medgen2efo.results.bindings[0].efo.value.replace(prefix, "");
//  return medgen2efo.results.bindings[0].efo.value;
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

SELECT ?assoc
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
      ?assoc terms:mapped_trait ?mapped_trait ;
        terms:mapped_trait_uri ?mapped_trait_uri .
    }
    ?study dct:identifier ?study_id ;
      dct:description ?description ;
      terms:initial_sample_size ?initial_sample_size ;
      terms:replication_sample_size ?replication_sample_size .
 }
}

```

## `rs2tgv`
```javascript
async ({efo2gwas, base_url}) => {
  let rs_uri = [];
  const tmp = [];
  const rs_array = {};
  const tgv_tag = {};
  let count = 0;
  const variant_and_risk_allele ={};
  
  efo2gwas.results.bindings.map(binding => {
    variant_and_risk_allele[binding.variant_and_risk_allele.value] = binding.variant_and_risk_allele.value;
    binding.variant_and_risk_allele.value.split("; ").forEach(rs => {
      rs_uri.push("<http://identifiers.org/dbsnp/" + rs.split('-')[0] + ">");
    });
  });
  rs_uri = Array.from(new Set(rs_uri));
  for (const elem of rs_uri) {
    tmp.push(elem);
    if (tmp.length == 100){
      let api = "https://test43.biosciencedbc.jp/sparqlist/api/rs2tgvid";
      rs_array[count] = api.concat("?ep=https://togovar-dev.biosciencedbc.jp/sparql&rs_uri=", tmp.join(" ")).toString();
      count ++;
      tmp.length = 0;
    }
    let api = "https://test43.biosciencedbc.jp/sparqlist/api/rs2tgvid";
    rs_array[count] = api.concat("?ep=https://togovar-dev.biosciencedbc.jp/sparql&rs_uri=", tmp.join(" ")).toString();
  };
  for (let key in rs_array){
    const tmp_res = await fetch(rs_array[key], {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });
    const json = await tmp_res.json();
    for (let binding of json.results.bindings){
      const rs = binding.dbsnp.value.replace("http://identifiers.org/dbsnp/","");
      tgv_tag[rs] = binding.tgv_id.value ? "(<a href='" + base_url + "/variant/" + binding.tgv_id.value + "'>" + binding.tgv_id.value +"</a>)" : "";
    }
  }
   for (let key in variant_and_risk_allele){
     const val = variant_and_risk_allele[key];
     const disp_rs = [];
     val.split("; ").forEach(rs => {
       const html = tgv_tag[rs.split('-')[0]] ? tgv_tag[rs.split('-')[0]] : "";
      disp_rs.push("<a href='https://www.ebi.ac.uk/gwas/variants/" + rs.split('-')[0] + "'>" + rs + "</a>" + html );
    });
     variant_and_risk_allele[key] = disp_rs.join();
   }
  return variant_and_risk_allele;  
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
async ({efo2gwas,rs2tgv,trait_html}) => {
  const associations = {};
  return efo2gwas.results.bindings.map(d => {
    if (! associations[d.assoc.value]){
      associations[d.assoc.value] = d.assoc.value;
      return {
        variant_and_risk_allele: rs2tgv[d.variant_and_risk_allele.value],
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
      };
    }
  });
}
```
