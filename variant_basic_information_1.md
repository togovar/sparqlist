# Variant basic information

## Parameters

* `tgv_id` TogoVar ID
  * default: 48140310

## Endpoint

http://togovar.l5dev.jp/sparql-test

## `result1` fetch basic information

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

## `so_id`

```javascript
({
  json({result1}) {
    var r = result1.results.bindings;
    return r[0].var_class.value;
  }
})
```

## `result2` fetch display term for variant_class

```sparql
DEFINE sql:select-option "order"

PREFIX obo: <http://purl.obolibrary.org/obo/>

SELECT ?label
FROM <http://togovar.org/graph/so>
WHERE {
  VALUES ?so { obo:{{so_id}} }
  ?so rdfs:label ?label .
}
```

## Output

```javascript
({
  json({result1,result2}) {
    var r1 = result1.results.bindings;
    var r2 = result2.results.bindings;
    var obj = [
                { header: 'TogoVar ID',
                    data: 'tgv' + r1[0].tgv_id.value },
                { header: 'rs number',
                    data: r1[0].rs ? r1[0].rs.value : '' },
                { header: 'Variation',
                    data: r2[0].label.value }
              ];

    return obj;
  }
})
```
