# Variant report / ClinVar

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 1-6475089-A-G
* `tgv_id` TogoVar ID
  * default:
  * example: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `tgv_id`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id}) => {
  if (variant.length > 0) {
    const url = SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/variant2tgv?variant=${encodeURIComponent(variant)}`);
    const res = await fetch(url);

    return await res.text();
  }

  if (tgv_id.length > 0) {
    return tgv_id
  }

  return 'not found'
}
```

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
      cvo:classified_record/cvo:classifications/cvo:germline_classification/cvo:review_status ?review_status ;
      cvo:classified_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?_rcv cvo:rcv_classifications/cvo:germline_classification/cvo:description/cvo:description ?interpretation ;
      cvo:rcv_classifications/cvo:germline_classification/cvo:description/cvo:date_last_evaluated ?last_evaluated ;
      cvo:classified_condition_list/cvo:classified_condition ?_classified_condition .

    ?_classified_condition rdfs:label ?condition ;
      dct:source ?db ;
      dct:identifier ?medgen .
    FILTER(?db IN ("MedGen"))
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
