# Variant report / Summary

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{ep}}

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX dct: <http://purl.org/dc/terms/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX obo_in_owl: <http://www.geneontology.org/formats/oboInOwl#>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX m2r: <http://med2rdf.org/ontology/med2rdf#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?variation ?label ?type_label ?so ?reference ?ref ?alt ?hgvs
FROM <http://togovar.biosciencedbc.jp/variation>
FROM <http://togovar.biosciencedbc.jp/so>
WHERE {
    VALUES ?tgv_id { "{{tgv_id}}" }

    ?variation dct:identifier ?tgv_id ;
        rdfs:label ?label ;
        faldo:location ?_loc ;
        a ?_type .

    ?_loc (faldo:end|faldo:after)?/faldo:reference ?reference .

    OPTIONAL { ?variation m2r:reference_allele ?ref . }
    OPTIONAL { ?variation m2r:alternative_allele ?alt . }

    FILTER ( ?_type IN (obo:SO_0001483, obo:SO_0000667, obo:SO_0000159, obo:SO_1000032, obo:SO_1000002) ) .

    OPTIONAL {
      ?_type rdfs:label ?type_label ;
        obo_in_owl:id ?_so_id .

      BIND(REPLACE(STR(?_so_id), ":", "_") AS ?so)
    }

    OPTIONAL {
       ?variation tgvo:hasConsequence/rdfs:label ?hgvs .
       FILTER(!STRSTARTS(?hgvs, 'ENS'))
    }
}
```
