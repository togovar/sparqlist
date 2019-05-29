# Variant report / Gene

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv47263127
* `assembly` assembly
  * default: GRCh37
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
FROM <http://togovar.biosciencedbc.jp/graph/variant>
FROM <http://togovar.biosciencedbc.jp/graph/{{graph}}>
FROM <http://togovar.biosciencedbc.jp/graph/hgnc>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }

  ?variant a ?_vt ;
    tgvo:hasConsequence ?_cn .

  FILTER ( ?_vt IN (obo:SO_0001483, obo:SO_0000667, obo:SO_0000159, obo:SO_1000032, obo:SO_1000002) ) .

  OPTIONAL { 
    ?_cn tgvo:gene ?gene . 
    OPTIONAL { ?gene rdfs:label ?symbol . }
    OPTIONAL { 
      ?gene rdfs:seeAlso ?gene_xref .
      FILTER ( strstarts(str(?gene_xref), "http://identifiers.org/hgnc/") ) .
      BIND ( IRI( REPLACE(str(?gene_xref), "HGNC:", "") ) AS ?hgnc )
      OPTIONAL { ?hgnc dct:description ?approved_name . }
      OPTIONAL { ?hgnc skos:altLabel ?synonym . }
    }
  }
}
```
