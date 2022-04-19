# Disease report / Header

## Parameters

* `medgen_cid` MedGen CID
  * default: C0023467
  * example: C2675520

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX medgen: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?approved_name
WHERE {
  VALUES ?medgen { medgen:{{medgen_cid}} }

  GRAPH <http://togovar.biosciencedbc.jp/medgen> {
    ?medgen rdfs:label ?approved_name .
  }
}
```
