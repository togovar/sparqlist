# Variant basic information

## Parameters

* `tgv_id` TogoVar ID
  * default: 48140310

## Endpoint

http://togovar.l5dev.jp/sparql-test

## `result` fetch basic information

```sparql
DEFINE sql:select-option "order"

PREFIX dc: <http://purl.org/dc/terms/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX cv: <http://purl.jp/bio/10/clinvar/>
PREFIX tgl: <http://togovar.org/lookup#>

SELECT ?tgv_id ?var_class ?rs
FROM <http://togovar.org/graph/lookup>
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  ?var dc:identifier ?tgv_id ;
    tgl:variant_class ?var_class .
  OPTIONAL { ?var tgl:rs ?rs . }
}
```

## Output

```javascript
({
  json({result}) {
    var r = result.results.bindings;
    var obj = [
                { header: 'TogoVar ID',
                    data: 'tgv' + r[0].tgv_id.value },
                { header: 'rs number',
                    data: r[0].rs ? r[0].rs.value : '' },
                { header: 'Variation',
                    data: r[0].var_class.value }
              ];

    return obj;
  }
})
```
