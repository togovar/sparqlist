# TogoVar variant report JBrowse position

Obtain genomic position with TogoVarID

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 30913364

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `result`

```sparql
DEFINE sql:select-option "order"
PREFIX tgvl: <http://togovar.org/lookup#>
PREFIX hco: <http://identifiers.org/hco/>

SELECT ?seq_label ?start ?stop
FROM <http://togovar.org/graph/lookup>
FROM <http://togovar.org/graph/hco>
FROM <http://togovar.org/graph/refseq>
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  ?var tgvl:chromosome ?chr ;
    tgvl:start ?start ;
    tgvl:stop ?stop .
  BIND(IRI(CONCAT("http://identifiers.org/hco/", ?chr, "#GRCh37")) as ?hco)
  ?hco hco:refseq/rdfs:label ?seq_label .
}
```