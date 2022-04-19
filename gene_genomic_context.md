# Gene report / Genomic context

## Parameters

* `hgnc_id` HGNC ID
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX hco:   <http://identifiers.org/hco/>
PREFIX hgnc:  <http://identifiers.org/hgnc/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?chromosome ?start ?stop
WHERE {
  VALUES ?hgnc { hgnc:{{ hgnc_id }} }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    ?hgnc rdfs:label ?symbol .
  }

  GRAPH <http://togovar.biosciencedbc.jp/ensembl> {
    ?ensg rdfs:label ?symbol ;
      faldo:location/faldo:begin/faldo:reference ?reference ;
      faldo:location/faldo:begin/faldo:position ?start ;
      faldo:location/faldo:end/faldo:position ?stop .

    BIND(REPLACE(REPLACE(STR(?reference), hco:, ""), "#.*", "") AS ?chromosome)
  }
}
```
