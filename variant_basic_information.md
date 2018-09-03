# Variant basic information

## Parameters

* `tgv_id` TogoVar ID
  * default: 48140310

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `result` fetch basic information

```sparql
DEFINE sql:select-option "order"

PREFIX dc: <http://purl.org/dc/terms/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX cv: <http://purl.jp/bio/10/clinvar/>
PREFIX tgl: <http://togovar.org/lookup#>

SELECT ?tgv_id ?variant_type ?rs ?chr ?start ?ref ?alt ?label ?condition ?significance ?hgvs
FROM <http://togovar.org/graph/lookup>
FROM <http://togovar.org/graph/so>
FROM <http://togovar.org/graph/hgnc>
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  ?var dc:identifier ?tgv_id ;
    tgl:variant_type ?variant_type ;
    tgl:chromosome ?chr ;
    tgl:start ?start ;
    tgl:stop ?stop ;
    tgl:hgvs_g ?hgvs .
  ?variant_type rdfs:label ?label .
  OPTIONAL { ?var tgl:clinvar/tgl:conditions ?condition . }
  OPTIONAL { ?var tgl:clinvar/tgl:significances ?significance . }
  OPTIONAL { ?var tgl:ref ?ref . }
  OPTIONAL { ?var tgl:alt ?alt . }
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
                { header: 'refSNP ID',
                    data: r[0].rs ? '<a href=\"https://www.ncbi.nlm.nih.gov/snp/' + r[0].rs.value.split('/').slice(-1)[0] + '\">' + r[0].rs.value.split('/').slice(-1)[0] + '</a>' : '' },
                { header: 'Variation',
                    data: r[0].label.value },
                { header: 'Chromosome',
                    data: r[0].chr.value },
                { header: 'Position',
                    data: r[0].start.value },
                { header: 'Reference allele',
                    data: r[0].ref ? r[0].ref.value : '' },
                { header: 'Alternative allele',
                    data: r[0].alt ? r[0].alt.value : '' },
                { header: 'Condition',
                    data: r[0].condition ? r.map(x => x.condition.value) : [] },
                { header: 'Clinical Significance',
                    data: r[0].significance ? Array.from(new Set(r.map(x => x.significance.value))).sort().join(" / ") : "" },
                { header: 'HGVS',
                    data: r[0].hgvs ? r[0].hgvs.value : '' }
              ];

    return obj;
  }
})
```
