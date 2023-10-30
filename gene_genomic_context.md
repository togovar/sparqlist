# Gene report / Genomic context

## Parameters

* `hgnc_id` HGNC ID
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `validated_hgnc_id`

```javascript
({hgnc_id})=>{
  if(hgnc_id.match(/^\d+$/)){
    return hgnc_id
  }
}
```

## `result`

```sparql
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX hco:   <http://identifiers.org/hco/>
PREFIX hgnc:  <http://identifiers.org/hgnc/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?chromosome ?start ?stop
WHERE {
  {{#if validated_hgnc_id}}
  VALUES ?hgnc { hgnc:{{validated_hgnc_id}} }

  GRAPH <http://togovar.org/hgnc> {
    ?hgnc rdfs:label ?symbol .
  }

  GRAPH <http://togovar.org/ensembl> {
    ?ensg rdfs:label ?symbol ;
      faldo:location/faldo:begin/faldo:reference ?reference ;
      faldo:location/faldo:begin/faldo:position ?start ;
      faldo:location/faldo:end/faldo:position ?stop .

    BIND(REPLACE(REPLACE(STR(?reference), hco:, ""), "#.*", "") AS ?chromosome)
  }
  {{/if}}
}
```
