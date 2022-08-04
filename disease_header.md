# Disease report / Header

## Parameters

* `medgen_cid` MedGen CID
  * default: C0023467
  * example: C2675520

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `validated_medgen_cid`

```javascript
({medgen_cid}) => {
  if (medgen_cid.match(/^CN?\d+$/)) {
    return medgen_cid
  }
}
```

## `result`

```sparql
PREFIX medgen: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?approved_name
WHERE {
  {{#if validated_medgen_cid}}
  VALUES ?medgen { medgen:{{validated_medgen_cid}} }

  GRAPH <http://togovar.biosciencedbc.jp/medgen> {
    ?medgen rdfs:label ?approved_name .
  }
  {{/if}}
}
```
