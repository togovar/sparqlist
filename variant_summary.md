# Variant report / Summary

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv47263127

## Endpoint

{{{ep}}}

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX m2r: <http://med2rdf.org/ontology/med2rdf#>

SELECT DISTINCT ?variant ?hgvs ?variant_type ?so_type ?reference ?start ?stop ?ref ?alt
FROM <http://togovar.biosciencedbc.jp/graph/variant>
FROM <http://togovar.biosciencedbc.jp/graph/so>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }
  {
    ?variant a ?_vt ;
      rdfs:label ?hgvs ;
      faldo:location ?_loc .

    FILTER ( ?_vt IN (obo:SO_0001483, obo:SO_0000667, obo:SO_0000159, obo:SO_1000032, obo:SO_1000002) ) .

    ?_vt rdfs:label ?variant_type .
    BIND(REPLACE(STR(?_vt), ".*/", "") AS ?so_type)

    ?_loc faldo:begin?/faldo:reference ?reference ;
      faldo:begin?/faldo:position ?start .
    OPTIONAL { ?_loc faldo:end/faldo:position ?stop . }

    OPTIONAL { ?variant m2r:reference_allele ?ref . }
    OPTIONAL { ?variant m2r:alternative_allele ?alt . }
  }
}
```
