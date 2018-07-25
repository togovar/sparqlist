# TogoVar variant report summary NanoStanza

Obtain summary of a TogoVar variant

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 30425514

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `variants` description

```sparql
DEFINE sql:select-option "order"
PREFIX tgv: <http://togovar.org/lookup#>

SELECT DISTINCT *
FROM <http://togovar.org/graph/lookup>
WHERE {
  <http://togovar.org/variation/{{tgv_id}}>
    tgv:chromosome ?chr ;
    tgv:start ?pos .
    OPTIONAL { <http://togovar.org/variation/{{tgv_id}}> tgv:ref ?ref . }
    OPTIONAL { <http://togovar.org/variation/{{tgv_id}}> tgv:alt ?alt . }
}
```

## Output

```javascript
({
  json({variants}) {
    let r = variants.results.bindings;
    if (typeof r[0]['ref'] === "undefined") {
      variants.results.bindings[0]['ref'] = {'type': 'literal', 'value': '-'}
    } else if (r[0]['ref'].value.length > 4) {
      variants.results.bindings[0]['ref'].value = r[0]['ref'].value.substr(0, 4) + '...' + '(' + r[0]['ref'].value.length + ')'
    }
    if (typeof r[0]['alt'] === "undefined") {
      variants.results.bindings[0]['alt'] = {'type': 'literal', 'value': '-'}
    } else if (r[0]['alt'].value.length > 4) {
      variants.results.bindings[0]['alt'].value = r[0]['alt'].value.substr(0, 4) + '...' + '(' + r[0]['alt'].value.length + ')'
    }
    return variants;
  }
})
```
