# TogoVar ID to dbSNP ID

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv47264307

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `tgv_id`

```javascript
async ({tgv_id}) => {
  const value = String(tgv_id || "").trim();
  const regex = /^tgv\d+$/;

  if (value.length === 0) {
    return "";
  }

  if (value.match(regex)) {
    return value;
  }

  throw new Error(`Invalid ID: ${tgv_id}`);
}
```

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?rs
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?variant rdfs:seeAlso ?rs .

    FILTER STRSTARTS(STR(?rs), 'http://identifiers.org/dbsnp/rs')
  }
}
```
