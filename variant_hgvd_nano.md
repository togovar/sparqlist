# TogoVar variant report HGVD NanoStanza

Obtain stats of a TogoVar variant

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 421843

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `results`

```sparql
DEFINE sql:select-option "order"
PREFIX tgv: <http://togovar.org/lookup#>

SELECT DISTINCT *
FROM <http://togovar.org/graph/lookup>
WHERE {
  <http://togovar.org/variation/{{tgv_id}}>
    tgv:hgvd/tgv:frequency ?freq ;
    tgv:hgvd/tgv:num_alt_alleles ?num_alt ;
    tgv:hgvd/tgv:num_alleles ?num_all .
}
```

## `json`

```javascript
({
  json({results}) {
    if (result = results.results.bindings[0]) {
      return {
        "title": "HGVD",
        "num_alt": result.num_alt.value,
        "num_all": result.num_all.value,
        "freq": result.freq.value
      };
    } else {
      return { "title": "HGVD" };
    }
  }
})
```

