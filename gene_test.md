# Variant report / Gene

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv47263126
* `assembly` assembly
  * default: GRCh38
  * example: GRCh37, GRCh38

## Endpoint

{{{ep}}}

## `graph`

```javascript
({
  json({assembly}) {
    return assembly === "GRCh38" ? "ensembl38" : "ensembl37" ;
  }
})
```

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX dct: <http://purl.org/dc/terms/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>

SELECT DISTINCT ?variant ?gene ?symbol ?approved_name ?synonym
FROM <http://togovar.biosciencedbc.jp/variant>
FROM <http://togovar.biosciencedbc.jp/{{graph}}>
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }

  ?variant a ?_vt ;
    tgvo:hasConsequence ?_cn .

  FILTER ( ?_vt IN (obo:SO_0001483, obo:SO_0000667, obo:SO_0000159, obo:SO_1000032, obo:SO_1000002) ) .

  OPTIONAL {
    ?_cn tgvo:gene ?gene .
    OPTIONAL {
      ?gene rdfs:label ?symbol .
    }
    OPTIONAL {
      ?gene rdfs:seeAlso/^rdfs:seeAlso ?hgnc .
      FILTER ( strstarts(str(?hgnc), "http://identifiers.org/hgnc/") ) .
      OPTIONAL { ?hgnc dct:description ?approved_name . }
      OPTIONAL { ?hgnc skos:altLabel ?synonym . }
    }
  }
}
```
