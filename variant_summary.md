# Variant report / Summary

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo: <http://genome-variation.org/resource#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?type ?variation ?reference ?ref ?alt ?hgvs
FROM <http://togovar.biosciencedbc.jp/variant>
FROM <http://togovar.biosciencedbc.jp/variant/annotation/ensembl>
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  ?variation dct:identifier ?tgv_id ;
    a ?_type ;
    faldo:location ?_loc ;
    gvo:ref ?ref ;
    gvo:alt ?alt ;
    faldo:location/(faldo:end|faldo:after)?/faldo:reference ?reference .

    BIND(REPLACE(STR(?_type), "http://genome-variation.org/resource#", "") AS ?type)
    BIND(IRI(CONCAT("http://identifiers.org/hco/", REPLACE(STR(?variation), "-.*", ""), "/GRCh37#", REPLACE(STR(?variation), "^[^-]+-", ""))) AS ?hco)

  OPTIONAL {
    ?hco tgvo:hasConsequence/tgvo:hgvsg ?hgvs .
  }
}
```
