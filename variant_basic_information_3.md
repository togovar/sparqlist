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

SELECT ?condition ?significance ?hgvs
FROM <http://togovar.org/graph/lookup>
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  ?var tgl:clinvar ?clinvar ;
    tgl:hgvs_g ?hgvs .
  ?clinvar tgl:conditions ?condition ;
    tgl:significances ?significance .
}
```

## Output

```javascript
({
  json({result}) {
    var r = result.results.bindings;
    var obj = [
                { header: 'Condition',
                    data: r[0].condition ? r.map(x => x.condition.value) : [] },
                { header: 'Significance',
                    data: r[0].significance ? r.map(x => x.significance.value) : [] },
                { header: 'HGVS',
                    data: r[0].hgvs ? r[0].hgvs.value : '' }
              ];

    return obj;
  }
})
```
