# TogoVar rs2disease stanza biblio query

Obtain biblio info by PubMed IDs

## Parameters

* `pmids` PubMed IDs
  * default: 26386261,26708094

## Endpoint

http://ep.dbcls.jp/sparql-togovar

## `values`

```javascript
({pmids}) => {
  return pmids.split(",").map(pmid => '"' + pmid + '"').join(" ")
}
```

## `results` statistics

```sparql
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>
PREFIX colil: <http://purl.jp/bio/10/colil/ontology/201303#>

SELECT ?pmid ?article_title ?source ?authors ?year ?citation_count ?mesh
WHERE {
  VALUES ?pmid  { {{values}} }
  ?article colil:Authors ?authors ;
           togows:pmid ?pmid ;
           togows:ti ?article_title ;
           togows:so ?source ;
           togows:dp ?year .
  BIND (0 as ?citation_count)
  BIND ("" as ?mesh)
}
ORDER BY DESC(?year)
```
