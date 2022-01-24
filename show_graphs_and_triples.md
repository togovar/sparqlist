# show_graphs_and_triples_forRDFportal

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql


## Endpoint

{{ep}}

## `hgnc`

```sparql
SELECT <http://togovar.biosciencedbc.jp/hgnc> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {?s ?p ?o}
}
```

## Endpoint

https://togovar-dev.biosciencedbc.jp/sparql

## `clinvar`

```sparql
SELECT <http://togovar.biosciencedbc.jp/clinvar> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/clinvar> {?s ?p ?o}
}
```

## Endpoint

https://togovar-dev.biosciencedbc.jp/sparql

## `medgen`

```sparql
SELECT <http://togovar.biosciencedbc.jp/medgen> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/medgen> {?s ?p ?o}
}
```

## `pubmed`

```sparql
SELECT <http://togovar.biosciencedbc.jp/pubmed> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/pubmed> {?s ?p ?o}
}
```

## `pubtator`

```sparql
SELECT <http://togovar.biosciencedbc.jp/pubtator> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/pubtator> {?s ?p ?o}
}
```


## `nlm_catalog`

```sparql
SELECT <http://togovar.biosciencedbc.jp/nlm-catalog> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/nlm-catalog> {?s ?p ?o}
}
```


## `ensembl`

```sparql
SELECT <http://togovar.biosciencedbc.jp/ensembl37> count(*) WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/ensembl37> {?s ?p ?o}
}
```

## `result`

```javascript
({hgnc, clinvar, medgen, pubmed, pubtator, nlm_catalog, ensembl}) =>{
  let result = {}
  let key1 = Object.keys(hgnc.results.bindings[0])[0]
  let key2 = Object.keys(hgnc.results.bindings[0])[1]
  let list = [hgnc, clinvar, medgen, pubmed, pubtator, nlm_catalog, ensembl]
  for (let x in list){
    result[list[x].results.bindings[0][key1].value] = list[x].results.bindings[0][key2].value
  }
  
  return result
}
```
