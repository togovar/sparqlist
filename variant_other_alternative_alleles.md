# Variant report / Other alternative alleles

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv55413188
  * example: tgv218973

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

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
async ({SPARQLIST_TOGOVAR_SEARCH, label}) => {
  const binding = label.results.bindings[0];

  if (binding) {
    const match = binding.label.value.match(/http:\/\/identifiers.org\/hco\/(.+)\/.+#(\d+)/);
    const position = `${match[1]}:${match[2]}`;
    const res = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat("?stat=0&quality=0&term=", position), {
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
