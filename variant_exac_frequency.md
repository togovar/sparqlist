# TogoVar variant report ExAC frequency

Obtain ExAC frequency

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 30913364

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `result`

```sparql
DEFINE sql:select-option "order"
PREFIX dc: <http://purl.org/dc/terms/>
PREFIX tgv: <http://togovar.org/lookup#>
PREFIX exo: <http://purl.jp/bio/10/exac/>

SELECT ?dataset ?population ?num_alt ?num_all ?freq
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  GRAPH <http://togovar.org/graph/tgexac> {
    ?var rdfs:seeAlso ?exac .
  }
  GRAPH <http://togovar.org/graph/exac> {
    ?exac a exo:Variant ;
      exo:population ?p .
    ?p rdfs:label ?population ;
      exo:alleleCount ?num_alt ;
      exo:alleleNum ?num_all ;
      exo:alleleFrequency ?freq .
      BIND("ExAC" AS ?dataset)
  }
}
ORDER BY ?population
```