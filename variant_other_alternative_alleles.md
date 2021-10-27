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

## `label`
```sparql
PREFIX dct: <http://purl.org/dc/terms/>

SELECT ?label
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variation dct:identifier ?tgv_id .
    BIND(STR(?variation) AS ?label)
  }
}
```

## `result`

```javascript
async ({search_api, label}) => {
  const binding = label.results.bindings[0];

  if (binding) {
    const position = `${binding.label.value.split('-')[0]}:${binding.label.value.split('-')[1]}`;
    const res = await fetch(search_api.concat("?stat=0&quality=0&term=", position), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });
    return res.json();
  } else {
    return { data: [] };
  }
}
```
