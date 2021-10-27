# Variant report / Gene

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{ep}}

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo: <http://genome-variation.org/resource#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?variation ?gene ?hgnc ?symbol ?approved_name ?synonym
WHERE {
  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    {
      SELECT DISTINCT ?variation ?hco ?gene ?hgnc
      WHERE {
        VALUES ?tgv_id { "{{tgv_id}}" }

        GRAPH <http://togovar.biosciencedbc.jp/variant> {
          ?variation dct:identifier ?tgv_id .
          BIND(IRI(CONCAT("http://identifiers.org/hco/", REPLACE(STR(?variation), "-.*", ""), "/GRCh37#", REPLACE(STR(?variation), "^[^-]-", ""))) AS ?hco)
        }

        GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/ensembl> {
          OPTIONAL {
            ?hco tgvo:hasConsequence/tgvo:gene ?gene .
            ?hco tgvo:hasConsequence/tgvo:hgnc ?hgnc .
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
