# Variant report / Summary

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv56616325
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 16-89196249-G-A

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
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX tgvo:  <http://togovar.org/vocabulary/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos:  <http://www.w3.org/2004/02/skos/core#>

SELECT DISTINCT ?type ?reference ?position ?ref ?alt ?gene ?hgnc ?symbol ?approved_name
WHERE {
  VALUES ?variant { <{{variant}}> }

  GRAPH <http://togovar.org/variant> {
    ?variant a ?class ;
      faldo:location/faldo:begin?/faldo:reference ?reference ;
      gvo:pos_vcf ?position ;
      gvo:ref_vcf ?ref ;
      gvo:alt_vcf ?alt .

    BIND (
      IF(?class = gvo:SNV, "SNV", 
        IF(?class = gvo:Insertion, "Insertion", 
          IF(?class = gvo:Deletion, "Deletion", 
            IF(?class = gvo:Indel, "Indel", 
              IF(?class = gvo:MNV, "Substitution", "Unknown")
            )
          )
        )
      ) AS ?type
    )
  }

  OPTIONAL {
    GRAPH <http://togovar.org/variant/annotation/ensembl> {
      ?variant tgvo:hasConsequence/tgvo:gene ?gene ;
        tgvo:hasConsequence/tgvo:hgnc ?hgnc .
      FILTER STRSTARTS(STR(?gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")
    }

    GRAPH <http://togovar.org/hgnc> {
      ?hgnc rdfs:label ?symbol ;
        dct:description ?approved_name .
    }
  }
}
```
