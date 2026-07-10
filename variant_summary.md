# Variant report / Summary

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 16-89196249-G-A
* `tgv_id` TogoVar ID
  * example: tgv56616325

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `tgv_id`

```javascript
// Accepts either a VCF-style `variant` (CHROM-POS-REF-ALT) or a `tgv_id`.
// When `variant` is given, resolve it to a `tgv_id` via variant2tgv first,
// so the `result` step below only ever has to deal with `tgv_id`.
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
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX tgvo:  <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?type ?reference ?position ?ref ?alt ?gene ?hgnc ?symbol ?approved_name
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    # `begin?` is a zero-or-one path: some variants attach faldo:reference
    # directly to faldo:location, others via an intermediate faldo:begin.
    ?variant a ?class ;
      dct:identifier ?tgv_id ;
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

  # Also resolve the overlapping gene (if any), so callers don't need a
  # separate request to variant_gene.md for the same variant.
  OPTIONAL {
    GRAPH <http://togovar.org/variant/annotation/ensembl> {
      ?variant tgvo:hasConsequence/tgvo:gene ?gene ;
        tgvo:hasConsequence/tgvo:hgnc ?hgnc .
      FILTER STRSTARTS(STR(?gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")
    }

    # Keep these as required triples (not nested OPTIONALs) here: nesting
    # OPTIONAL inside this block made the production endpoint time out
    # (20s+); requiring both together resolves in ~0.1s.
    GRAPH <http://togovar.org/hgnc> {
      ?hgnc rdfs:label ?symbol ;
        dct:description ?approved_name .
    }
  }
}
```
