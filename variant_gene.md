# Variant report / Gene

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 1-6475089-A-G
* `tgv_id` TogoVar ID
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
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>

SELECT DISTINCT ?variant ?gene ?hgnc ?symbol ?approved_name ?synonym
WHERE {
  GRAPH <http://togovar.org/hgnc> {
    {
      SELECT DISTINCT ?variant ?hco ?gene ?hgnc
      WHERE {
        VALUES ?tgv_id { "{{tgv_id}}" }

        GRAPH <http://togovar.org/variant> {
          ?variant dct:identifier ?tgv_id .
        }

        GRAPH <http://togovar.org/variant/annotation/ensembl> {
          OPTIONAL {
            ?variant tgvo:hasConsequence/tgvo:gene ?gene ;
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
