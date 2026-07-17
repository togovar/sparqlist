# Variant report / Transcript

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv219804
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 1-6475089-A-G

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `variant`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id}) => {
  let params;
  if (tgv_id.length > 0) {
    params = `tgv_id=${encodeURIComponent(tgv_id)}`;
  } else if (variant.length > 0) {
    params = `variant=${encodeURIComponent(variant)}`;
  } else {
    throw new Error("Either tgv_id or variant must be provided.");
  }

  const res = await fetch(SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/resolve_variant?${params}`));

  if (!res.ok) {
    throw new Error((await res.text()).replace(/^Error: /, ""));
  }

  return await res.text();
}
```

## `result`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX dc11: <http://purl.org/dc/elements/1.1/>
PREFIX tgvo: <http://togovar.org/vocabulary/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?transcript ?enst_id ?gene_symbol ?gene_xref ?hgvs_p ?hgvs_c ?sift ?polyphen ?alpha_missense
                (GROUP_CONCAT(DISTINCT ?_consequence_label ; separator = ",") AS ?consequence_label)
WHERE {
  VALUES ?variant { <{{variant}}> }

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
    }
    OPTIONAL {
      ?_consequence tgvo:hgvsc ?hgvs_c .
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
      ?_consequence tgvo:gene_symbol ?gene_symbol ;
                    tgvo:gene_symbol_source ?_gene_symbol_source .
      FILTER ( ?_gene_symbol_source IN ("HGNC", "EntrezGene") )
    }
    OPTIONAL {
      ?_consequence tgvo:hgnc ?gene_xref .
    }
  }
}
ORDER BY ?transcript
```
