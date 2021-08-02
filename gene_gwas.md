# Gene report / GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `hgnc_id` HGNC ID
  * default: 404

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

## `gene2gwas`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT ?variant_and_risk_allele
       ?raf
       ?p_value
       ?odds_ratio
       ?ci_text
       ?beta
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

## `result`

```javascript
({gene2gwas}) => {
  return gene2gwas.results.bindings.map(d => ({
    variant_and_risk_allele: d.variant_and_risk_allele.value,
    raf: d.raf.value,
    p_value: d.p_value.value,
    odds_ratio: d.odds_ratio.value,
    ci_text: d.ci_text.value,
    beta: d.beta.value,
    mapped_trait: d.mapped_trait.value,
    mapped_trait_uri: d.mapped_trait_uri.value,
    pubmed_id: d.pubmed_id.value,
    pubmed_uri: d.pubmed_uri.value,
    study_detail: d.study_detail.value,
    study: d.study.value,
    initial_sample_size: d.initial_sample_size.value,
    replication_sample_size: d.replication_sample_size.value
  }));
}
```
