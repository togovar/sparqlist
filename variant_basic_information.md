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

## `result2` fetch basic information

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

## `result3` fetch basic information

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
  json({result1,result2,result3}) {
    var r1 = result1.results.bindings;
    var r2 = result2.results.bindings;
    var r3 = result3.results.bindings;

    var obj = [];

    obj.push({ header: 'TogoVar ID',
                 data: 'tgv' + r1[0].tgv_id.value },
             { header: 'rs number',
                 data: r1[0].rs ? r1[0].rs.value : '' },
             { header: 'Variation',
                 data: r1[0].var_class.value });

    obj.push({ header: 'Chromosome',
                 data: r2[0].chr.value },
             { header: 'Position',
                 data: r2[0].start.value },
             { header: 'Gene',
                 data: r2[0].symbol ? r2[0].symbol.value : '' },
             { header: 'Reference',
                 data: r2[0].ref ? r2[0].ref.value : '' },
             { header: 'Alternative',
                 data: r2[0] ? r2[0].alt.value : '' });

    obj.push({ header: 'Condition',
                 data: r3[0].condition ? r3.map(x => x.condition.value) : [] },
             { header: 'Significance',
                 data: r3[0].significance ? r3.map(x => x.significance.value) : [] },
             { header: 'HGVS',
                 data: r3[0].hgvs ? r3[0].hgvs.value : '' });

    return obj;
  }
})
```
