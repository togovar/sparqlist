# Variant report / Other alternative alleles

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 7-127614533-G-A
* `tgv_id` TogoVar ID
  * default:
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

## `result`

```sparql
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>

SELECT ?reference ?start ?stop
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    {
      ?variant a gvo:SNV ;
        dct:identifier ?tgv_id ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:position ?start .
    } UNION {
      ?variant a gvo:Insertion ;
        dct:identifier ?tgv_id ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:before ?start .
    } UNION {
      ?variant a gvo:Deletion ;
        dct:identifier ?tgv_id ;
        faldo:location/faldo:begin/faldo:reference ?reference ;
        faldo:location/faldo:begin/faldo:before ?start ;
        faldo:location/faldo:end/faldo:after ?stop .
    } UNION {
      ?variant a gvo:Indel ;
        dct:identifier ?tgv_id ;
        faldo:location/faldo:begin/faldo:reference ?reference ;
        faldo:location/faldo:begin/faldo:before ?start ;
        faldo:location/faldo:end/faldo:after ?stop .
    } UNION {
      ?variant a gvo:MNV ;
        dct:identifier ?tgv_id ;
        faldo:location/faldo:reference ?reference ;
        faldo:location/faldo:begin ?start ;
        faldo:location/faldo:end ?stop .
    }
  }
}
```

## `result`

```javascript
async ({result, SPARQLIST_TOGOVAR_APP}) => {
  const binding = result.results.bindings[0];

  if (binding) {
    const match = binding.reference.value.match(/http:\/\/identifiers.org\/hco\/(.+)\//);
    let region = `${match[1]}:${binding.start.value}`;
    if (binding.stop) {
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
