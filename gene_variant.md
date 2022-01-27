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
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }
  ?hgnc_uri rdfs:label ?symbol.
}
```

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_SEARCH, id2symbol}) => {
  let binding = id2symbol.results.bindings[0].symbol.value;
  const max_rows = 10000;

  if (binding) {
    const first_res = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat("?stat=0&quality=0&term=", binding), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });
    const first_json = await first_res.json();
    let result_data = first_json.data;
    const data_count = first_json.statistics.filtered < max_rows ? first_json.statistics.filtered : max_rows;

    let tmp_res = [];
    let counnt = 0;

    for (let i = 1; i * 100 < data_count; i++) {
      let offset = i * 100;
      let res = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat("?stat=0&quality=0&term=", binding, "&offset=", offset), {
        method: 'GET',
        headers: {
          'Accept': 'application/json',
        },
      });
      let json = await res.json();
      tmp_res[i] = json.data;
      count = i;
    }

    for (let j = 1; j <count + 1; j++){
     result_data = result_data.concat(tmp_res[j]);
    }
    return  { data: result_data };
  } else {
    return { data: [] };
  }
}
```