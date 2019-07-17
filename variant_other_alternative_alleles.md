# Variant report / Other alternative alleles

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv55413188
  * example: tgv218973
* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `search_api` Search endpoint
  * default: https://togovar.biosciencedbc.jp/search

## Endpoint

{{ep}}

## `position`
```sparql
DEFINE sql:select-option "order"

PREFIX tgv:   <http://togovar.biosciencedbc.jp/variant/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX hco:   <http://identifiers.org/hco/>

SELECT (IF(BOUND(?stop), CONCAT(?chromosome, ":", ?start, "-", ?stop), CONCAT(?chromosome, ":", ?start))) AS ?position
FROM <http://togovar.biosciencedbc.jp/graph/variant>
WHERE {
  VALUES ?variant { tgv:{{tgv_id}} }

  ?variant faldo:location/faldo:begin?/faldo:reference ?reference .
  ?variant faldo:location/faldo:begin?/faldo:position ?start .
  OPTIONAL { ?variant faldo:location/faldo:end/faldo:position ?stop . }

  BIND( REPLACE(REPLACE(STR(?reference), hco:, ""), "#.*", "") AS ?chromosome )
}
```

## `result`

```javascript
async ({search_api, position}) => {
  let binding = position.results.bindings[0];

  if (binding) {
    let res = await fetch(search_api.concat("?stat=0&quality=0&term=", binding.position.value));
    return res.json();
  } else {
    return { data: [] };
  }
}
```
