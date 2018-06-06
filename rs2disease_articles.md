# TogoVar rs2disease stanza articles query

Obtain PubMed IDs by dbSNP ID

## Parameters

* `rs` dbSNP ID
  * default: rs864309721
  * examples: rs4917014

## Endpoint

http://ep.dbcls.jp/sparql-togovar

## `results` rs# to pmid#

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>

SELECT ?article
WHERE {
   ?node rdf:type oa:Annotation ;
         oa:hasTarget ?article ;
         oa:hasBody dbsnp:{{rs}} .
}
LIMIT 20
```
