# Disease report / GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `medgen_cid` MedGen CID
  * default: C0023467 
  * example: C0023467(acute myeloid leukemia), C2675520(breast-ovarian cancer)

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
#FROM <http://togovar.biosciencedbc.jp/medgen>
FROM <http://togovar.biosciencedbc.jp/efo>
FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
WHERE {
 VALUES ?mapped_trait_uri { <{{ efo }}> }

 GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
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
}
```

## `result`

```javascript
({efo2gwas}) => {
  return efo2gwas.results.bindings.map(d => ({
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
