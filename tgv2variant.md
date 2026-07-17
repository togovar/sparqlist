# TogoVar ID to Variant

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv47264307

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `tgv_id`

```javascript
async ({tgv_id}) => {
  const regex = /^tgv\d+$/;
  const match = tgv_id.match(regex);

  if (match) {
    return tgv_id;
  } else {
    throw new Error(`Invalid ID: ${tgv_id}`);
  }
}
```

## `variant`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?variant
FROM <http://togovar.org/variant>
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  ?variant dct:identifier ?tgv_id .
}
LIMIT 1
```

## `result`

```javascript
async ({variant}) => {
  const bindings = variant.results.bindings

  if (bindings[0]) {
    return bindings[0].variant.value;
  } else {
    throw new Error(`Variant not found`);
  }
}
```
