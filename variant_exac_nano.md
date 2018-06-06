# TogoVar variant report ExAC NanoStanza

Obtain stats of a TogoVar variant

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 421843

## Endpoint

https://togovar.org/sparql

## `results`

```sparql
DEFINE sql:select-option "order"
PREFIX tgv: <http://togovar.org/lookup#>

SELECT DISTINCT *
FROM <http://togovar.org/graph/lookup>
WHERE {
  <http://togovar.org/variation/{{tgv_id}}>
    tgv:exac/tgv:frequency ?freq ;
    tgv:exac/tgv:num_alt_alleles ?num_alt ;
    tgv:exac/tgv:num_alleles ?num_all .
}
```

## `json`

```javascript
({
  json({results}) {
    if (result = results.results.bindings[0]) {
      return { 
        "title": "ExAC",
        "num_alt": result.num_alt.value,
        "num_all": result.num_all.value,
        "freq": result.freq.value
      };
    } else {
      return { "title": "ExAC" };
    }
  }
})
```
