# Variant report / Other alternative alleles

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv55413188
  * example: tgv218973

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>

SELECT ?reference ?start ?stop
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
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
async ({result, SPARQLIST_TOGOVAR_SEARCH}) => {
  const binding = result.results.bindings[0];

  if (binding) {
    const match = binding.reference.value.match(/http:\/\/identifiers.org\/hco\/(.+)\//);
    let region = `${match[1]}:${binding.start.value}`;
    if (binding.stop) {
      region += `-${binding.stop.value}`;
    }

    const res = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat("?stat=0&quality=0&term=", encodeURIComponent(region)), {
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
