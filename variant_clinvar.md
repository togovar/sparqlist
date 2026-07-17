# Variant report / ClinVar

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv219804
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 1-6475089-A-G

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `variant`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id}) => {
  let params;
  if (tgv_id.length > 0) {
    params = `tgv_id=${encodeURIComponent(tgv_id)}`;
  } else if (variant.length > 0) {
    params = `variant=${encodeURIComponent(variant)}`;
  } else {
    throw new Error("Either tgv_id or variant must be provided.");
  }

  const res = await fetch(SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/resolve_variant?${params}`));

  if (!res.ok) {
    throw new Error((await res.text()).replace(/^Error: /, ""));
  }

  return await res.text();
}
```

## `result`

```sparql
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?title ?vcv_review_status ?rcv_review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv ?rcv
WHERE {
  VALUES ?variant { <{{variant}}> }

  GRAPH <http://togovar.org/variant/annotation/clinvar> {
    ?variant dct:identifier ?variation_id .

    BIND(IRI(CONCAT("http://ncbi.nlm.nih.gov/clinvar/variation/", ?variation_id)) AS ?clinvar)
  }

  GRAPH <http://togovar.org/clinvar> {
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:classified_record/cvo:classifications/cvo:germline_classification/cvo:review_status ?vcv_review_status ;
      cvo:classified_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?_rcv cvo:accession ?rcv ;
          cvo:rcv_classifications/cvo:germline_classification [ 
            cvo:description/cvo:description ?interpretation ;
            cvo:review_status ?rcv_review_status 
          ] ;
          cvo:classified_condition_list/cvo:classified_condition ?_classified_condition .

    OPTIONAL {
      ?_rcv cvo:rcv_classifications/cvo:germline_classification/cvo:description/cvo:date_last_evaluated ?last_evaluated .
    }

    ?_classified_condition rdfs:label ?condition ;
      dct:source ?db ;
      dct:identifier ?medgen .
    FILTER(?db IN ("MedGen"))
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
