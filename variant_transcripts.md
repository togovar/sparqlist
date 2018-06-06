# Variant transcripts

## Parameters

* `tgv_id` TogoVar ID
  * default: 48140310

## Endpoint

https://togovar.l5dev.jp/sparql

## `result` fetch transcripts

```sparql
PREFIX tgl: <http://togovar.org/lookup#>
SELECT ?transcript ?enst_id ?so_label ?ensg_id ?symbol ?symbol_source ?ncbi_gene ?hgvs_c str(?sift) AS ?sift str(?polyphen) as ?polyphen
FROM <http://togovar.org/graph/lookup>
FROM <http://togovar.org/graph/so>
WHERE {
  {
    SELECT DISTINCT ?transcript
    FROM <http://togovar.org/graph/lookup>
    WHERE {
      VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
      ?var tgl:transcript ?transcript .
    }
  }
  OPTIONAL { ?transcript tgl:ensembl_transcript ?enst_id . }
  OPTIONAL { ?transcript tgl:consequence/rdfs:label ?so_label . }
  OPTIONAL { ?transcript tgl:ensembl_gene ?ensg_id . }
  OPTIONAL { ?transcript tgl:symbol ?symbol . }
  OPTIONAL { ?transcript tgl:symbol_source ?symbol_source . }
  OPTIONAL { ?transcript tgl:ncbi_gene ?ncbi_gene . }
  OPTIONAL { ?transcript tgl:hgvs_c ?hgvs_c . }
  OPTIONAL { ?transcript tgl:sift ?sift . }
  OPTIONAL { ?transcript tgl:polyphen ?polyphen . }
}
```

## Output

```javascript
({
  json({result}) {
    var groupBy = function(xs, key) {
      return xs.reduce(function(rv, x) {
        (rv[x[key]] = rv[x[key]] || []).push(x);
        return rv;
      }, {});
    };
    
    var formatJson = (obj) => {
      return obj.map(x => {
        var o = {}
        Object.keys(x).forEach(y => o[y] = x[y]['value']);
        return o;
      })
    }
    
    var obtainId = (uri) => {
      if (uri) {
        var arr = uri.match(/([^\/.]+)/g);
        return arr[arr.length - 1];
      } else {
        return '';
      }
    }

    var r1 = result.results.bindings;    
    var r2 = formatJson(r1)
    var r3 = groupBy(r2, 'transcript')

    var header = [
      'Transcript ID',
      // 'Gene ID',
      'Gene symbol',
      // 'HGVS',
      'Consequence',
      'SIFT',
      'Polyphen',
    ];

    var data = []
    Object.keys(r3).forEach(x => {
      var row = [];
      row.push('<a href=\"https://identifiers.org/ensembl/' + obtainId(r3[x][0]['enst_id']) + '\">' + obtainId(r3[x][0]['enst_id']) + '</a>');
      // row.push(obtainId(r3[x][0]['ensg_id']));
      if (r3[x][0]['symbol']) {
        row.push('<a href=\"https://www.genenames.org/cgi-bin/gene_symbol_report?match=' + r3[x][0]['symbol'] + '\">' + r3[x][0]['symbol'] + '</a>');
      } else {
        row.push('');
      }
      // row.push(r3[x][0]['hgvs_c']);
      row.push(r3[x].map(y => y['so_label']));
      row.push(r3[x][0]['sift'] || '');
      row.push(r3[x][0]['polyphen'] || '');
      data.push(row);
    })

    return {
      'header': header,
      'data': data
    };
  }
})
```
