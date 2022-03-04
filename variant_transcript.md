# Variant report / Transcript

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX dc11: <http://purl.org/dc/elements/1.1/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?transcript ?enst_id ?gene_symbol ?gene_xref (GROUP_CONCAT(DISTINCT ?_consequence_label ; separator = ",") AS ?consequence_label) ?hgvs_p ?hgvs_c ?sift ?polyphen
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variation dct:identifier ?tgv_id .
    BIND(IRI(CONCAT("http://identifiers.org/hco/", REPLACE(STR(?variation), "-.*", ""), "/GRCh37#", REPLACE(STR(?variation), "^[^-]+-", ""))) AS ?hco)
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/ensembl> {
    ?hco tgvo:hasConsequence ?_consequence .
    ?_consequence a ?_consequence_type .

    GRAPH <http://togovar.biosciencedbc.jp/so> {
      ?_consequence_type rdfs:label ?_consequence_label .
    }
    OPTIONAL { ?_consequence tgvo:sift ?sift . }
    OPTIONAL { ?_consequence tgvo:polyphen ?polyphen . }
    OPTIONAL {
      ?_consequence tgvo:hgvsp ?hgvs_p .
      FILTER STRSTARTS(?hgvs_p, 'ENSP')
    }
    OPTIONAL {
      ?_consequence tgvo:hgvsc ?hgvs_c .
      FILTER STRSTARTS(?hgvs_c, 'ENST')
    }
    OPTIONAL {
      ?_consequence tgvo:transcript ?transcript .
      OPTIONAL {
        GRAPH <http://togovar.biosciencedbc.jp/ensembl> {
          ?transcript dct:identifier|dc11:identifier ?enst_id . 
        }
      }
    }
    OPTIONAL {
      ?_consequence tgvo:gene ?_gene .

      FILTER STRSTARTS(STR(?_gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")

      OPTIONAL {
        GRAPH <http://togovar.biosciencedbc.jp/ensembl> {
          ?_gene rdfs:label ?gene_symbol .
        }
      }
    }
    OPTIONAL {
      ?_consequence tgvo:hgnc ?gene_xref .
    }
  }
}
ORDER BY ?transcript
```
