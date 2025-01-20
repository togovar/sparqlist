# Variant report / Transcript

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * default: 1-6475089-A-G
* `tgv_id` TogoVar ID
  * default:
  * example: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `tgv_id`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id}) => {
  if (variant.length > 0) {
    const url = SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/variant2tgv?variant=${encodeURIComponent(variant)}`);
    const res = await fetch(url);

    return await res.text();
  }

  if (tgv_id.length > 0) {
    return tgv_id
  }

  return 'not found'
}
```

## `result`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX dc11: <http://purl.org/dc/elements/1.1/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?transcript ?enst_id ?gene_symbol ?gene_xref ?hgvs_p ?hgvs_c ?sift ?polyphen
                ?alpha_missense
                (GROUP_CONCAT(DISTINCT ?_consequence_label ; separator = ",") AS ?consequence_label)
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?variant tgvo:hasConsequence ?_consequence .
    ?_consequence a ?_consequence_type .

    GRAPH <http://togovar.org/so> {
      ?_consequence_type rdfs:label ?_consequence_label .
    }
    OPTIONAL { ?_consequence tgvo:sift ?sift . }
    OPTIONAL { ?_consequence tgvo:polyphen ?polyphen . }
    OPTIONAL { ?_consequence tgvo:alphamissense ?alpha_missense . }
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
        GRAPH <http://togovar.org/ensembl> {
          ?transcript dct:identifier|dc11:identifier ?enst_id .
        }
      }
    }
    OPTIONAL {
      ?_consequence tgvo:gene ?_gene .

      FILTER STRSTARTS(STR(?_gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")

      OPTIONAL {
        GRAPH <http://togovar.org/ensembl> {
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
