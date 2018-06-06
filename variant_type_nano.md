# TogoVar Variant Type NanoStanza

Obtain variant type of a TogoVar variant

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 6332

## Endpoint

https://togovar.org/sparql

## `variant_type`

```sparql
DEFINE sql:select-option "order"
PREFIX tgv: <http://togovar.org/lookup#>

SELECT DISTINCT ?variant_type
FROM <http://togovar.org/graph/lookup>
FROM <http://togovar.org/graph/so>
WHERE {
  <http://togovar.org/variation/{{tgv_id}}>
    tgv:variant_type/rdfs:label ?variant_type .
}
```
