# TogoVar ID to dbSNP ID

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv47264307

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?rs
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?variant rdfs:seeAlso ?rs .

    FILTER STRSTARTS(STR(?rs), 'http://identifiers.org/dbsnp/rs')
  }
}
```
