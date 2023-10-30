# Variant report / ClinVar

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?title ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  GRAPH <http://togovar.org/variant/annotation/clinvar> {
    ?variant dct:identifier ?variation_id .

    BIND(IRI(CONCAT("http://ncbi.nlm.nih.gov/clinvar/variation/", ?variation_id)) AS ?clinvar)
  }

  GRAPH <http://togovar.org/clinvar> {
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?_rcv cvo:interpretation ?interpretation ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .

    ?_interpreted_condition rdfs:label ?condition ;
      dct:source ?db ;
      dct:identifier ?medgen .
    FILTER(?db IN ("MedGen"))
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
