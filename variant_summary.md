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

SELECT DISTINCT ?type ?reference ?start ?stop ?ref ?alt ?hgvs
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    {
      ?variant a gvo:SNV ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:position ?start .
      BIND("SNV" AS ?type)
    } UNION {
      ?variant a gvo:Insertion ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:after ?start ;
        faldo:location/faldo:before ?stop .
      BIND("Insertion" AS ?type)
    } UNION {
      ?variant a gvo:Deletion ;
        faldo:location/faldo:begin/faldo:reference ?reference ;
        faldo:location/faldo:begin/faldo:before ?start ;
        faldo:location/faldo:end/faldo:after ?stop .
      BIND("Deletion" AS ?type)
    } UNION {
      ?variant a gvo:Indel ;
        faldo:location/faldo:begin/faldo:reference ?reference ;
        faldo:location/faldo:begin/faldo:before ?start ;
        faldo:location/faldo:end/faldo:after ?stop .
      BIND("Indel" AS ?type)
    } UNION {
      ?variant a gvo:MNV ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:begin ?start ;
        faldo:location/faldo:end ?stop .
      BIND("Substitution" AS ?type)
    }

    ?variant dct:identifier ?tgv_id ;
      gvo:ref ?ref ;
      gvo:alt ?alt .
  }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    OPTIONAL {
      ?variant tgvo:hasConsequence/tgvo:hgvsg ?hgvs .
    }
  }
}
```
