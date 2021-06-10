# Gene / variant

## Parameters

* `hgnc_id` HGNC ID
  * default: 404
* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `search_api` Search endpoint
  * default: https://togovar.biosciencedbc.jp/search

## Endpoint
{{ ep }}

## `id2symbol`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
SELECT DISTINCT ?symbol
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }
  ?hgnc_uri rdfs:label ?symbol.
}

```

## `result`

```javascript
async ({search_api, id2symbol}) => {
  let binding = id2symbol.results.bindings[0].symbol.value;

  if (binding) {
    let res = await fetch(search_api.concat("?stat=0&quality=0&term=", binding));
    return res.json();
  } else {
    return { data: [] };
  }
}
```
