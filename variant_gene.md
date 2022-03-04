# Variant report / Gene

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX tgvo:  <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos:  <http://www.w3.org/2004/02/skos/core#>

SELECT DISTINCT ?variation ?gene ?hgnc ?symbol ?approved_name ?synonym
WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    {
      SELECT DISTINCT ?variation ?hco ?gene ?hgnc
      WHERE {
        VALUES ?tgv_id { "{{tgv_id}}" }

        GRAPH <http://togovar.biosciencedbc.jp/variant> {
          ?variation dct:identifier ?tgv_id .
        }

        GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/ensembl> {
          OPTIONAL {
            ?variation tgvo:hasConsequence/tgvo:gene ?gene ;
                       tgvo:hasConsequence/tgvo:hgnc ?hgnc .
            FILTER STRSTARTS(STR(?gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")
          }
        }    
      }
    }

    OPTIONAL {
      ?hgnc rdfs:label ?symbol .
    }
    OPTIONAL {
      ?hgnc dct:description ?approved_name .
    }
    OPTIONAL {
      ?hgnc skos:altLabel ?synonym .
    }
  }
}
```
