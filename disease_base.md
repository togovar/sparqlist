# Disease report / Base 

## Parameters

* `medgen_cid` MedGen CID
  * default: C2675520 
  * example: C2675520
* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql

## Endpoint

{{ ep }}

## `medgen`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX medgen: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

SELECT ?medgen_cid ?label ?definition
FROM <http://togovar.biosciencedbc.jp/medgen>
WHERE {
  VALUES ?medgen { medgen:{{ medgen_cid }} }
  
  GRAPH <http://togovar.biosciencedbc.jp/medgen>{
    ?medgen dct:identifier ?medgen_cid.
    ?medgen rdfs:label ?label.
    ?medgen skos:definition ?definition.
  }
}
```

## `result`

```javascript
({medgen}) => {
  return medgen.results.bindings.map(d => ({
    medgen_cid: d.medgen_cid.value,
    label: d.label.value,
    definition: d.definition.value
  }));
}
```
