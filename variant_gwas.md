# Variant_GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv47264307

## Endpoint

{{ ep }}


## `tgv2rs`
```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX sio: <http://semanticscience.org/resource/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX exo: <http://purl.jp/bio/10/exac/>

SELECT ?tgv_id ?label ?variation ?rs_uri
FROM <http://togovar.biosciencedbc.jp/variation>
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
  let prefix = "http://identifiers.org/dbsnp/";
  return tgv2rs.results.bindings[0].rs_uri.value.replace(prefix, "");
}
```

## `rs2gwas`
```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#> 
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/> 
PREFIX gwas: <http://rdf.ebi.ac.uk/terms/gwas/> 
PREFIX oban: <http://purl.org/oban/> 
PREFIX owl: <http://www.w3.org/2002/07/owl#> 
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#> 
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#> 
PREFIX ro: <http://www.obofoundry.org/ro/ro.owl#> 
PREFIX study: <http://www.ebi.ac.uk/gwas/studies/> 
PREFIX dct: <http://purl.org/dc/terms/> 
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#> 
PREFIX pubmed: <http://rdf.ncbi.nlm.nih.gov/pubmed/> 

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
       VALUES ?rs_id  { "{{ rs_str }}" }
       ?assoc a gwas:Association ;
          terms:reported_genes ?reported_genes;  # 'ALDH2''' ;
          terms:strongest_snp_risk_allele ?variant_and_risk_allele;
          terms:risk_allele_frequency ?raf;
          terms:snps ?rs_id;  #  "rs671" ;  
          terms:p_value ?p_value;  # 2E-70
          terms:odds_ratio ?odds_ratio;   #0.22 
          terms:beta ?beta;  #  "NA" ;
          terms:ci_text ?ci_text;  # "[0.2-0.24] unit increase" ;
          terms:mapped_trait ?mapped_trait;   #"sweet liking measurement" ;
          terms:mapped_trait_uri ?mapped_trait_uri ;  # <http://www.ebi.ac.uk/efo/EFO_0010156> ;
          terms:study ?study;  #  study:GCST010572 ;
          dct:date ?association_date;  #   "2020-09-16"^^xsd:date ;
          dct:references ?pubmed;   #pubmed:32572145 ;
          gwas:has_pubmed_id ?pubmed_id.  # "32572145" .
      
      ?study dct:identifier ?study_id;  #"GCST010426" ;
          dct:description ?description;  #"Systolic blood pressure x educational attainment (some college) interaction (2df)"@en ;
          terms:initial_sample_size ?initial_sample_size;  # "27,617 European ancestry individuals with some college education, 20,253 E
          terms:replication_sample_size ?replication_sample_size.  # "131,584 European ancestry individuals with some college education
} ORDER by ?p_value
```

## `result`
```javascript
({rs2gwas})=>{
  return rs2gwas.results.bindings.map(d=>{ 
    return {
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
    };
  });	
}
```



