# Gene report / Base

## Parameters

* `hgnc_id` HGNC Id
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX obo:  <http://purl.obolibrary.org/obo/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

SELECT ?hgnc_uri ?gene_symbol ?approved_name ?chromosomal_location ?alias ?ncbigene ?ensg ?refseq ?omim
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{hgnc_id}}> }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    ?hgnc_uri rdfs:label ?gene_symbol ;
      dct:description ?approved_name ;
      obo:so_part_of ?chromosomal_location .

    OPTIONAL {
      ?hgnc_uri skos:altLabel ?alias .
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?ncbigene .
      FILTER STRSTARTS(STR(?ncbigene), "http://identifiers.org/ncbigene/")
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?refseq .
      FILTER STRSTARTS(STR(?refseq), "http://identifiers.org/refseq/")
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?omim.
      FILTER STRSTARTS(STR(?omim), "http://identifiers.org/omim/")
    }
    OPTIONAL {
      ?hgnc_uri rdfs:seeAlso ?ensg .
      FILTER STRSTARTS(STR(?ensg), "http://identifiers.org/ensembl/ENSG")
    }
  }
}
```
