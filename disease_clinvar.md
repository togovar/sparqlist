# Disease report / Clinvar

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `medgen_cid` MedGen CID 
  * default: C0023467
  * example: C0023467

## Endpoint

{{ ep }}

## `medgen_clinvar`

```sparql
PREFIX cvo: <http://purl.jp/bio/10/clinvar/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX medgen: <http://ncbi.nlm.nih.gov/medgen/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?vcv_disp ?clinvar ?interpretation ?review_status ?last_evaluated ?condition
FROM <http://togovar.biosciencedbc.jp/clinvar>
FROM <http://togovar.biosciencedbc.jp/variation/annotation/clinvar>
FROM <http://togovar.biosciencedbc.jp/variation>
WHERE {
  VALUES ?medgen { medgen:{{ medgen_cid }} }

  GRAPH <http://togovar.biosciencedbc.jp/clinvar>{
    ?_interpreted_condition dct:references ?medgen;
        rdfs:label ?condition .
    
    ?_rcv cvo:interpretation ?interpretation ;
      dct:identifier ?rcv ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .
    
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv.

    BIND (REPLACE (STR(?vcv), 'VCV0+', '') AS ?vcv_disp)
  }

  GRAPH <http://togovar.biosciencedbc.jp/variation/annotation/clinvar>{
    ?togovar tgvo:condition / rdfs:seeAlso ?clinvar .
  }

  GRAPH <http://togovar.biosciencedbc.jp/variation>{
    ?togovar dct:identifier ?tgv_id .
  }

}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```

## `result`

```javascript
({medgen_clinvar}) => {
  return medgen_clinvar.results.bindings.map(d => ({
    tgv_id: d.tgv_id.value,
    vcv: d.vcv_disp.value,
    clinvar: d.clinvar.value,
    interpretation: d.interpretation.value,
    review_status: d.review_status.value,
    last_evaluated: d.last_evaluated.value,
    condition: d.condition.value
  }));
}
```


