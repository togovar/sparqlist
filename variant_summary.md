# Variant report / Summary

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX tgvo:  <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?type ?variant ?reference ?ref ?alt ?hgvs
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variant a ?_type ;
      dct:identifier ?tgv_id ;
      faldo:location ?_loc ;
      gvo:ref ?ref ;
      gvo:alt ?alt ;
      faldo:location/(faldo:end|faldo:after)?/faldo:reference ?reference .

      BIND(REPLACE(STR(?_type), "http://genome-variation.org/resource#", "") AS ?type)
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/ensembl> {
    OPTIONAL {
      ?variant tgvo:hasConsequence/tgvo:hgvsg ?hgvs .
    }
  }
}
```
