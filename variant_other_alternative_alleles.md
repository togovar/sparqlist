# Variant report / Other alternative alleles

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 7-127614533-G-A
* `tgv_id` TogoVar ID
  * example: tgv30913364

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

## `search_range`

```sparql
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>

SELECT DISTINCT ?reference ?start ?stop
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id ;
      faldo:location/faldo:begin?/faldo:reference ?reference ;
      gvo:pos_vcf ?position ;
      gvo:ref_vcf ?ref .

    BIND ( ?position AS ?start )
    BIND ( (?position + STRLEN(?ref) - 1) AS ?stop )
  }
}
```

## `result`

```javascript
async ({search_range, SPARQLIST_TOGOVAR_APP}) => {
  const binding = search_range.results.bindings[0];

  if (binding) {
    const match = binding.reference.value.match(/http:\/\/identifiers.org\/hco\/(.+)\//);
    let region = `${match[1]}:${binding.start.value}`;
    if (binding.stop && binding.stop.value != binding.start.value) {
      region += `-${binding.stop.value}`;
    }

    const res = await fetch(SPARQLIST_TOGOVAR_APP.concat("/search?stat=0&quality=0&term=", encodeURIComponent(region)), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });

    return res.json();
  } else {
    return {data: []};
  }
}
```
