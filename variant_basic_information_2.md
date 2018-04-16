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

SELECT ?chr ?start ?ref ?alt ?symbol
FROM <http://togovar.org/graph/lookup>
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  ?var tgl:chromosome ?chr ;
    tgl:start ?start ;
    tgl:stop ?stop .
  OPTIONAL { ?var tgl:ref ?ref . }
  OPTIONAL { ?var tgl:alt ?alt . }
  OPTIONAL { ?var tgl:symbol ?symbol . }
}
```

## Output

```javascript
({
  json({result}) {
    var r = result.results.bindings;
    var obj = [
                { header: 'Chromosome',
                    data: r[0].chr.value },
                { header: 'Position',
                    data: r[0].start.value },
                { header: 'Gene',
                    data: r[0].symbol ? r[0].symbol.value : '' },
                { header: 'Reference',
                    data: r[0].ref ? r[0].ref.value : '' },
                { header: 'Alternative',
                    data: r[0] ? r[0].alt.value : '' },
              ];

    return obj;
  }
})
```
