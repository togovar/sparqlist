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

SELECT DISTINCT ?type ?reference ?position ?ref ?alt
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
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
}
```
