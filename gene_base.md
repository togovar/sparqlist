# Gene report / Base

## Parameters

* `hgnc_id`HGNC Id
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX obo: <http://purl.obolibrary.org/obo/>

SELECT ?hgnc_uri ?gene_symbol ?approved_name ?chromosomal_location ?alias ?ncbigene ?ensg ?refseq ?omim
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }

  ?hgnc_uri rdfs:label ?gene_symbol ;
    dct:description ?approved_name ;
    obo:so_part_of ?chromosomal_location .
    OPTIONAL { ?hgnc_uri skos:altLabel ?alias . }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?ncbigene .
      FILTER ( strstarts(str(?ncbigene), "http://identifiers.org/ncbigene/") ) .
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?refseq .
      FILTER ( strstarts(str(?refseq), "http://identifiers.org/refseq/") ) .
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?omim.
      FILTER ( strstarts(str(?omim), "http://identifiers.org/omim/") ) .
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?ensg .
      FILTER ( strstarts(str(?ensg), "http://identifiers.org/ensembl/ENSG") ) .
    }
}
```
