# Variant report / GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv47264307
  * example: tgv6252547

## Endpoint

{{ ep }}


## `tgv2rs`
```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?tgv_id ?label ?variation ?rs_uri
FROM <http://togovar.biosciencedbc.jp/variant>
WHERE {
  VALUES ?tgv_id { "{{ tgv_id }}" }

  ?variation dct:identifier ?tgv_id ;
    rdfs:label ?label;
    rdfs:seeAlso ?rs_uri.
}
```

## `rs_str`
```javascript
({tgv2rs}) => {
  const prefix = "http://identifiers.org/dbsnp/";
  return tgv2rs.results.bindings[0].rs_uri.value.replace(prefix, "");
}
```

## `rs2gwas`
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
       ?pubmed_id
       ?pubmed as ?pubmed_uri
       ?description as ?study_detail
       ?study ?initial_sample_size ?replication_sample_size
FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
WHERE {
  VALUES ?rs_id  { "{{ rs_str }}" }

  ?assoc a gwas:Association ;
    terms:reported_genes ?reported_genes;
    terms:strongest_snp_risk_allele ?variant_and_risk_allele;
    terms:risk_allele_frequency ?raf;
    terms:snps ?rs_id;
    terms:p_value ?p_value;
    terms:odds_ratio ?odds_ratio;
    terms:beta ?beta;
    terms:beta_unit ?beta_unit ;
    terms:ci_text ?ci_text;
    terms:study ?study;
    dct:date ?association_date;
    dct:references ?pubmed;
    gwas:has_pubmed_id ?pubmed_id.

  ?study dct:identifier ?study_id;
    dct:description ?description;
    terms:initial_sample_size ?initial_sample_size;
    terms:replication_sample_size ?replication_sample_size.

} ORDER by ?p_value
```

## `rs2traits`
```sparql
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT ?assoc ?mapped_trait ?mapped_trait_uri
WHERE{
  VALUES ?rs_id  { "{{ rs_str }}" }
  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
    ?assoc a gwas:Association ;
      terms:snps ?rs_id;
      terms:mapped_trait_uri ?mapped_trait_uri.
  }
  GRAPH <http://togovar.biosciencedbc.jp/efo>{
    ?mapped_trait_uri rdfs:label ?mapped_trait.
  }
}
```

## `trait_html`
```javascript
({rs2traits})=>{
  const traits = {};
  rs2traits.results.bindings.map(x=>{
    if (! traits[x.assoc.value]){ traits[x.assoc.value] = [] }
    traits[x.assoc.value].push('<a href="' + x.mapped_trait_uri.value + '">' + x.mapped_trait.value + '</a>');
  })
  return traits;
}
```

## `result`
```javascript
({rs2gwas, trait_html})=>{
  return rs2gwas.results.bindings.map(d=>{
    return {
      variant_and_risk_allele: d.variant_and_risk_allele.value,
      rs_uri: "https://www.ebi.ac.uk/gwas/variants/" + d.rs_id.value,
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
  });
}
```