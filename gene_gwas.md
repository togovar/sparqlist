# Gene report / GWAS

## Parameters

* `hgnc_id` HGNC ID
  * default: 404
* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `search_api` Search endpoint
  * default: https://togovar.biosciencedbc.jp/search
* `base_url` TogoVar URL
  * default: https://togovar-dev.biosciencedbc.jp
  
  
## Endpoint

{{ ep }}

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

## `search_api`
```javascript
async ({search_api, gene_symbol}) => {
  const max_rows = 10000;

  if (gene_symbol) {
    const first_res = await fetch(search_api.concat("?stat=0&quality=0&term=", gene_symbol), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });
    const first_json = await first_res.json();
    let result_data = first_json.data;
    const data_count = first_json.statistics.filtered < max_rows ? first_json.statistics.filtered : max_rows;
    
    let tmp_res = [];
    let counnt = 0;

    for (let i = 1; i * 100 < data_count; i++) {
      let offset = i * 100;
      let res = await fetch(search_api.concat("?stat=0&quality=0&term=", gene_symbol, "&offset=", offset), {
        method: 'GET',
        headers: {
          'Accept': 'application/json',
        },
      });
      let json = await res.json();
      tmp_res[i] = json.data;
      count = i;
    }

    for (let j = 1; j <count + 1; j++){
     result_data = result_data.concat(tmp_res[j]);
    }
    return  { data: result_data };
  } else {
    return { data: [] };
  }
}
```
## `gene2gwas`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT ?assoc
       ?variant_and_risk_allele
       ?rs_id
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
    terms:mapped_trait ?mapped_trait ;
    terms:mapped_trait_uri ?mapped_trait_uri ;
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

SELECT ?assoc ?mapped_trait ?mapped_trait_uri
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
## `extract_data`
```javascript
async ({search_api}) => {
  const res = {};
  search_api.data.map(a => {
    if(a.existing_variations !== void 0){
      freq = {};
      a.frequencies.forEach((elem,num) => {
        freq[elem.source] = elem.allele ; 
      });
      a.existing_variations.forEach((elem, num) => {
        res[elem] = {
          tgv_id: a.id,
          position: a.chromosome + ": " + a.start,
          ref_alt: '<div class="ref-alt"><span class="ref" data-sum="' + a.reference.length + '">' + a.reference + '</span><span class="arrow"></span><span class="alt" data-sum="' + a.alternative.length + '">' + a.alternative + '</span></div>',
          alt_freq: freq["gem_j_wga"]
        }
      });
    }
  });
  return res;
}
```

## `result`
```javascript
async ({gene2gwas, trait_html, extract_data, base_url}) => {
  const associations = {};
  const res = [];
  gene2gwas.results.bindings.map(d => {
    if (! associations[d.assoc.value]){
      associations[d.assoc.value] = d.assoc.value;
      res.push({
        tgv_id: extract_data[d.rs_id.value] ? extract_data[d.rs_id.value].tgv_id : "",
        tgv_link: extract_data[d.rs_id.value] ? base_url + "/variant/" + extract_data[d.rs_id.value].tgv_id : "",
        variant_and_risk_allele: d.variant_and_risk_allele.value,
        rs_uri: "https://www.ebi.ac.uk/gwas/variants/" + d.rs_id.value,
        position: extract_data[d.rs_id.value] ? extract_data[d.rs_id.value].position : "",
        ref_alt: extract_data[d.rs_id.value] ? extract_data[d.rs_id.value].ref_alt : "",
        alt_freq: extract_data[d.rs_id.value] ? extract_data[d.rs_id.value].alt_freq.frequency : "",
        raf: d.raf.value != "NR" ? parseFloat(d.raf.value) : null,
        p_value: d.p_value.value != "NAN" ? parseFloat(d.p_value.value) : null,
        odds_ratio: d.odds_ratio.value != "NA" ? parseFloat(d.odds_ratio.value) : null,
        ci_text: d.ci_text.value,
        beta: d.beta.value != "NA" ? parseFloat(d.beta.value) : null,
        beta_unit: d.beta_unit.value,
        mapped_trait: trait_html[d.assoc.value].join(),
        pubmed_id: d.pubmed_id.value,
        pubmed_uri: d.pubmed_uri.value,
        study_detail: d.study.value.replace("http://www.ebi.ac.uk/gwas/studies/",""),
        study: d.study.value,
        initial_sample_size: d.initial_sample_size.value,
        replication_sample_size: d.replication_sample_size.value
      });
    }
  });
  return res;
}
```
