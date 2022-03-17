# Gene report / Variant

## Parameters

* `hgnc_id` HGNC ID
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `id2symbol`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?symbol
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{hgnc_id}}> }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    ?hgnc_uri rdfs:label ?symbol .
  }
}
```

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_SEARCH, id2symbol}) => {
  const symbol = id2symbol.results.bindings[0]?.symbol?.value;
  const max_rows = 10000;

  if (symbol) {
    const r = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat(`?stat=0&quality=0&limit=0&term=${encodeURIComponent(symbol)}`), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    }).then(res => res.json());

    const total = r.statistics.filtered;

    return await [...Array(Math.ceil(Math.min(total, max_rows) / 100)).keys()].reduce(async (previousValue, currentValue) => {
      const prev = await previousValue;
      const data = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat(`?stat=0&quality=0&offset=${currentValue * 100}&term=${encodeURIComponent(symbol)}`), {
        method: 'GET',
        headers: {
          'Accept': 'application/json',
        },
      }).then(res => res.json()).then(json => json.data);

      return [...prev, data];
    }, []);
  }

  return [];
}
```
