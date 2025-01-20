# Variant report / Summary

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * default: 16-89196249-G-A
* `tgv_id` TogoVar ID
  * default:
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
PREFIX tgvo:  <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?type ?reference ?start ?stop ?ref ?alt ?hgvs
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    {
      ?variant a gvo:SNV ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:position ?start .
      BIND("SNV" AS ?type)
    } UNION {
      ?variant a gvo:Insertion ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:after ?start ;
        faldo:location/faldo:before ?stop .
      BIND("Insertion" AS ?type)
    } UNION {
      ?variant a gvo:Deletion ;
        faldo:location/faldo:begin/faldo:reference ?reference ;
        faldo:location/faldo:begin/faldo:before ?start ;
        faldo:location/faldo:end/faldo:after ?stop .
      BIND("Deletion" AS ?type)
    } UNION {
      ?variant a gvo:Indel ;
        faldo:location/faldo:begin/faldo:reference ?reference ;
        faldo:location/faldo:begin/faldo:before ?start ;
        faldo:location/faldo:end/faldo:after ?stop .
      BIND("Indel" AS ?type)
    } UNION {
      ?variant a gvo:MNV ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:begin ?start ;
        faldo:location/faldo:end ?stop .
      BIND("Substitution" AS ?type)
    }

    ?variant dct:identifier ?tgv_id ;
      gvo:ref ?ref ;
      gvo:alt ?alt .
  }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    OPTIONAL {
      ?variant tgvo:hasConsequence/tgvo:hgvsg ?hgvs .
    }
  }
}
```
