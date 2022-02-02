# Gene report / GWAS

## Parameters

* `hgnc_id` HGNC ID
  * default: 404
  
## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}


## `id2symbol`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?gene_symbol
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }
  ?hgnc_uri rdfs:label ?gene_symbol .
}
```

## `gene_symbol`

```javascript
({id2symbol}) => {
  return id2symbol.results.bindings[0].gene_symbol.value;
}
```

## `gene2gwas`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT DISTINCT ?assoc
       ?variant_and_risk_allele
       ?rs_id
       ?raf
       ?p_value
       ?odds_ratio
       ?ci_text
       ?beta
       ?beta_unit
       ?pubmed_id
       ?pubmed as ?pubmed_uri
       ?description as ?study_detail
       ?study
       ?initial_sample_size
       ?replication_sample_size
FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
WHERE {
  VALUES ?reported_genes  { "{{ gene_symbol }}" }
  ?assoc a gwas:Association ;
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

  ?study dct:identifier ?study_id ;
    dct:description ?description ;
    terms:initial_sample_size ?initial_sample_size ;
    terms:replication_sample_size ?replication_sample_size .
}
```

## `gene2traits`
```sparql
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/> 
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/> 
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#> 

SELECT DISTINCT ?assoc ?mapped_trait ?mapped_trait_uri
WHERE{
  VALUES ?reported_genes  { "{{ gene_symbol }}" }
  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
    ?assoc a gwas:Association ;
      terms:reported_genes ?reported_genes ;
      terms:mapped_trait_uri ?mapped_trait_uri.
  }
  GRAPH <http://togovar.biosciencedbc.jp/efo>{
    ?mapped_trait_uri rdfs:label ?mapped_trait.
  }
}
```

## `trait_html`
```javascript
({gene2traits})=>{
  const traits = {};
  gene2traits.results.bindings.map(x=>{
    if (! traits[x.assoc.value]){ traits[x.assoc.value] = [] } 
    traits[x.assoc.value].push('<a href="' + x.mapped_trait_uri.value + '">' + x.mapped_trait.value + '</a>');
  })
  return traits;
}
```

## `get_variant_info`
```javascript
 async ({SPARQLIST_TOGOVAR_SEARCH, gene2gwas}) => {
  let results = {};
  let rs_ids = new Set();

  gene2gwas.results.bindings.forEach(d => {
    rs_ids.add(d.rs_id.value);
  });

  for(var rs_id of rs_ids){
    const options = {
      method: 'GET',
      headers: {
         'Accept': 'application/json',
       }
    };
    const request_uri = SPARQLIST_TOGOVAR_SEARCH.concat("?stat=0&quality=0&term=", rs_id);
    try {
      await fetch(request_uri, options).then(res=>res.json()).then(json=> {
        json.data.forEach (d => {
          let gem_j_freq = null;
          d.frequencies.forEach (f => {
          if(f.source === 'gem_j_wga' && typeof f.allele != 'undefined'){
            gem_j_freq = f.allele.frequency;
         }
        });
          results[rs_id] = {
            tgv_id: d.id,
            position: d.chromosome + ":" + d.start,
            ref_alt: '<div class="ref-alt"><span class="ref" data-sum="' + d.reference.length + '">' + d.reference + '</span><span class="arrow"></span><span class="alt" data-sum="' + d.alternative.length + '">' + d.alternative + '</span></div>',
            alt_freq_gem_j_wga: gem_j_freq,
            raf: d.raf
          }
        });
      });
    }catch(e){
      console.log("Error in TogoVar search API for rs_id=" + rs_id);
    }
  }

  return results;
}
```

## `result`
```javascript
({gene2gwas, trait_html, get_variant_info}) => {
  const res = [];
  const variant_info = get_variant_info;
  gene2gwas.results.bindings.map(d => {
      res.push({
        tgv_id: variant_info[d.rs_id.value] ? variant_info[d.rs_id.value].tgv_id : "",
        tgv_link: variant_info[d.rs_id.value] ? "/variant/" + variant_info[d.rs_id.value].tgv_id : "",
        variant_and_risk_allele: d.variant_and_risk_allele.value,
        rs_uri: "https://www.ebi.ac.uk/gwas/variants/" + d.rs_id.value,
        position: variant_info[d.rs_id.value] ? variant_info[d.rs_id.value].position : "",
        ref_alt: variant_info[d.rs_id.value] ? variant_info[d.rs_id.value].ref_alt : "",
        alt_freq: variant_info[d.rs_id.value] ? variant_info[d.rs_id.value].alt_freq_gem_j_wga : "",
        raf: d.raf.value != "NR" ? parseFloat(d.raf.value) : null,
        p_value: d.p_value.value != "NAN" ? parseFloat(d.p_value.value) : null,
        odds_ratio: d.odds_ratio.value != "NA" ? parseFloat(d.odds_ratio.value) : null,
        ci_text: d.ci_text.value,
        beta: d.beta.value != "NA" ? parseFloat(d.beta.value) : null,
        beta_unit: d.beta_unit.value,
        mapped_trait: trait_html[d.assoc.value] ? trait_html[d.assoc.value].join() : "",
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