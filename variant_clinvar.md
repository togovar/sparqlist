# Variant report / ClinVar

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
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT ?title ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv
FROM <http://togovar.biosciencedbc.jp/variation>
FROM <http://togovar.biosciencedbc.jp/variation/annotation/clinvar>
FROM <http://togovar.biosciencedbc.jp/clinvar>
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  ?variation dct:identifier ?tgv_id ;
             tgvo:condition ?_condition .

  ?_condition dct:source "ClinVar" ;
              rdfs:seeAlso ?clinvar .

  ?clinvar a cvo:VariationArchiveType ;
          rdfs:label ?title ;
          cvo:accession ?vcv ;
          cvo:interpreted_record/cvo:review_status ?review_status ;
          cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

  ?_rcv cvo:interpretation ?interpretation ;
        cvo:date_last_evaluated ?last_evaluated ;
        cvo:interpreted_condition ?_interpreted_condition .

  ?_interpreted_condition rdfs:label ?condition .

  OPTIONAL {
    ?_interpreted_condition dct:source ?db ;
                            dct:identifier ?medgen .
    FILTER (?db IN ("MedGen"))
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
