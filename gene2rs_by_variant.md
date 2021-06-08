# Variant report / Gene

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `hgnc_gene_id` 
  * default: 404

## Endpoint

{{{ep}}}

## `gene2rsid`

```sparql
PREFIX hgnc: <http://identifiers.org/hgnc/HGNC_>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

select distinct ?rs_id where {
    VALUES ?hgnc_uri { hgnc:{{ hgnc_gene_id }} }
    ?rs_id ^rdfs:seeAlso / tgvo:hasConsequence / tgvo:gene / rdfs:seeAlso ?hgnc_uri .
}
```
