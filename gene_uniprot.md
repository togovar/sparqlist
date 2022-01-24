# Gene report / Uniprot

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `gene_symbol` gene name
  * default: ALDH2

## Endpoint

{{{ep}}}

## `result`

```sparql

DEFINE sql:select-option "order"

PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX ebi: <http://rdf.ebi.ac.uk/resource/ensembl/>
PREFIX ide: <http://identifiers.org/>
PREFIX dc: <http://purl.org/dc/elements/1.1/>
PREFIX se: <http://semanticscience.org/resource/>

SELECT DISTINCT ?gene ?uniprot
FROM <http://togovar.biosciencedbc.jp/graph/variation>
FROM <http://togovar.biosciencedbc.jp/graph/ensembl38>
WHERE {
  ?gene rdfs:label "{{ gene_symbol }}" ;
    rdfs:seeAlso ?_identifiers .

  ?_identifiers rdf:type ide:uniprot;
    se:SIO_000671/se:SIO_000300 ?uniprot .
}
```
