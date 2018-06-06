# TogoVar rs2disease stanza annotation query

Obtain disease MeSH IDs by PubMed IDs

## Parameters

* `pmids` PubMed IDs
  * default: 26386261,26708094

## Endpoint

http://ep.dbcls.jp/sparql-togovar

## `values`

```javascript
({pmids}) => {
  return pmids.split(",").map(pmid => "pmid:" + pmid).join(" ")
  //return pmids.split(",").map(pmid => '"' + pmid + '"').join(" ")
}
```

## `results` statistics

```sparql
#DEFINE sql:select-option "order"

PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX pmid: <http://identifiers.org/pubmed/>

#SELECT ?pmid ?article ?annotation ?resource
SELECT ?article ?annotation ?resource
WHERE {
  VALUES ?article { {{values}} }
  #VALUES ?pmid { {{values}} }
  #BIND (IRI(CONCAT("http://identifiers.org/pubmed/", ?pmid)) AS ?article)
  ?node rdf:type oa:Annotation ;
        oa:hasTarget ?article ;
        oa:hasBody ?annotation ;
        dcterms:source ?resource .
  FILTER (STRSTARTS(STR(?annotation), "http://identifiers.org/mesh/"))
}
```
