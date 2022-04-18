# Gene report / Header

## Parameters

* `hgnc_id` HGNC Id
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?xref ?approved_name
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{hgnc_id}}> }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    ?hgnc_uri rdfs:label ?xref;
      dct:description ?approved_name.
  }
}
```
