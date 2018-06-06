# TogoVar rs2disease stanza MeSH label query

Obtain MeSH label by Mesh IDs

## Parameters

* `meshs` MeSH IDs
  * default: D002658,C564525,C564525,D013921,D002658,D008209,D054868,D013921

## Endpoint

http://lsd.dbcls.jp/sparql

## `values`

```javascript
({meshs}) => {
  return meshs.split(",").map(mesh => "mesh:" + mesh).join(" ")
}
```

## `results` statistics

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX lsd: <http://purl.jp/bio/10/lsd/ontology/201209#>
PREFIX mesh: <http://purl.jp/bio/10/lsd/mesh/>

SELECT ?mesh (STR(?ja) AS ?ja_label) (STR(?en) AS ?en_label)
WHERE {
   VALUES ?mesh { {{values}} }
   ?mesh rdfs:label ?ja .
   FILTER(LANG(?ja) = "ja")
   ?mesh rdfs:label ?en .
   FILTER(LANG(?en) = "en")
}
```
