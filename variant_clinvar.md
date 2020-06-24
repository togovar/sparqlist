# Variant report / ClinVar

## Parameters

* `ep` Endpoint
  * default: https://togovar-stg.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv48208323

## Endpoint

{{{ep}}}

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>

SELECT ?title ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv
FROM <http://togovar.biosciencedbc.jp/graph/variant>
FROM <http://togovar.biosciencedbc.jp/graph/variant/condition/clinvar>
FROM <http://togovar.biosciencedbc.jp/graph/clinvar>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }

  ?variant tgvo:hasInterpretedCondition ?clinvar .

  ?clinvar a cvo:VariationArchiveType ;
    rdfs:label ?title ;
    cvo:accession ?vcv ;
    cvo:interpreted_record/cvo:review_status ?review_status ;
    cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

  ?_rcv cvo:interpretation ?interpretation ;
    cvo:date_last_evaluated ?last_evaluated ;
    cvo:interpreted_condition/cvo:type_rcv_interpreted_condition ?condition .

  OPTIONAL {
    ?_rcv cvo:interpreted_condition/cvo:db ?db .
    ?_rcv cvo:interpreted_condition/cvo:id ?medgen .
    FILTER( ?db IN ("MedGen") )
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
