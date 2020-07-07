# Variant report / ClinVar

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv48208323

## Endpoint

{{{ep}}}

## `tgv2clinvar`
```sparql
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>

SELECT ?clinvar
FROM <http://togovar.biosciencedbc.jp/graph/variant/condition/clinvar>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }
    ?variant tgvo:hasInterpretedCondition ?clinvar .
}
```


## `clinvarid`
```javascript

({tgv2clinvar}) => {
  let prefix = "http://identifiers.org/clinvar:";
  
  return tgv2clinvar.results.bindings.map(x => x.clinvar.value.replace(prefix, ""));
}
```

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX dc: <http://purl.org/dc/terms/>

SELECT ?title ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv
FROM <http://togovar.biosciencedbc.jp/graph/clinvar>
WHERE {
  VALUES ?clinvar { <http://ncbi.nlm.nih.gov/clinvar/variation/{{clinvarid}}> }

  GRAPH <http://togovar.biosciencedbc.jp/graph/clinvar> {  
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status .

    ?clinvar cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?_rcv cvo:interpretation ?interpretation ;
      cvo:date_last_evaluated ?last_evaluated  ;
      cvo:interpreted_condition/rdfs:label ?condition .

    OPTIONAL {
      ?_rcv cvo:interpreted_condition/dc:source ?db .
      ?_rcv cvo:interpreted_condition/dc:identifier ?medgen .
      FILTER( ?db IN ("MedGen") )
    }
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
