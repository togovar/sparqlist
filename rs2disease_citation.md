# TogoVar rs2disease stanza citation query

Obtain citations from Colil by PubMed IDs

## Parameters

* `pmids` PubMed IDs
  * default: 26386261,26708094

## Endpoint

http://colil.dbcls.jp/sparql

## `values`

```javascript
({pmids}) => {
  return pmids.split(",").map(pmid => '"' + pmid + '"').join(" ")
}
```

## `results` statistics

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX colil: <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{values}} }
  ?citation_paper bibo:cites ?reference_paper .
  ?reference_paper rdfs:seeAlso ?dummy .
  ?dummy rdf:type colil:PubMed ;
         togows:pmid ?pmid .
}
```
