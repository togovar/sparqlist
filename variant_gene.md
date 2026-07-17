# Variant report / Gene

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv47264307
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 12-111803962-G-A

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
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX tgvo:  <http://togovar.org/vocabulary/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos:  <http://www.w3.org/2004/02/skos/core#>

SELECT DISTINCT ?variant ?gene ?hgnc ?symbol ?approved_name ?synonym
WHERE {
  VALUES ?variant { <{{variant}}> }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?variant tgvo:hasConsequence/tgvo:gene ?gene ;
      tgvo:hasConsequence/tgvo:hgnc ?hgnc .
    FILTER STRSTARTS(STR(?gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")
  }

  GRAPH <http://togovar.org/hgnc> {
    ?hgnc rdfs:label ?symbol ;
      dct:description ?approved_name .

    OPTIONAL {
      ?hgnc skos:altLabel ?synonym .
    }
  }
}
```
