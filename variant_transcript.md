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

## `sparql_result`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX dc11: <http://purl.org/dc/elements/1.1/>
PREFIX tgvo: <http://togovar.org/vocabulary/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?transcript ?enst_id ?gene_symbol ?gene_xref ?hgvs_p ?hgvs_c ?sift ?polyphen ?alpha_missense ?cadd_phred
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
    OPTIONAL { ?_consequence tgvo:cadd_phred ?cadd_phred . }
    OPTIONAL { ?_consequence tgvo:hgvsp ?hgvs_p . }
    OPTIONAL { ?_consequence tgvo:hgvsc ?hgvs_c . }
    OPTIONAL {
      ?_consequence tgvo:transcript ?_transcript .
      OPTIONAL {
        GRAPH <http://togovar.org/ensembl> {
          ?_transcript dct:identifier|dc11:identifier ?enst_id .
        }
      }
    }
    BIND(
      COALESCE(
        ?_transcript,
        URI(CONCAT("https://www.ncbi.nlm.nih.gov/nuccore/", STRBEFORE(STR(?hgvs_c), ":")))
      ) AS ?transcript
    )
    OPTIONAL {
      ?_consequence tgvo:gene_symbol ?gene_symbol ;
                    tgvo:gene_symbol_source ?_gene_symbol_source .
      FILTER (?_gene_symbol_source IN ("HGNC", "EntrezGene"))
    }
    OPTIONAL {
      ?_consequence tgvo:hgnc ?gene_xref .
    }
  }
}
ORDER BY ?transcript
```

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_APP, variant, sparql_result}) => {
  const match = String(variant).match(/^http:\/\/identifiers\.org\/hco\/(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)\/(?<reference>GRCh3[78])#(?<pos>\d+)-(?<ref>.+)-(?<alt>.+)$/);

  if (!match || !match.groups) {
    return sparql_result;
  }

  const position = Number.parseInt(match.groups.pos, 10);
  if (!Number.isFinite(position)) {
    return sparql_result;
  }

  if (!SPARQLIST_TOGOVAR_APP) {
    return sparql_result;
  }

  let data;
  try {
    const res = await fetch(SPARQLIST_TOGOVAR_APP.concat("/api/search/variant?stat=0&data=1"), {
      method: "POST",
      headers: {
        Accept: "application/json",
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        query: {
          variant: {
            chromosome: match.groups.chr,
            position: position,
            reference: match.groups.ref,
            alternate: match.groups.alt
          }
        },
        offset: 0,
        limit: 1
      })
    });

    if (!res.ok) {
      return sparql_result;
    }

    const json = await res.json();
    data = Array.isArray(json.data) ? json.data[0] : null;
  } catch (error) {
    return sparql_result;
  }

  const maneSelectTranscripts = new Set(
    (data?.transcripts || [])
      .filter(transcript => transcript.mane_select === true)
      .map(transcript => transcript.transcript_id)
      .filter(Boolean)
  );

  if (maneSelectTranscripts.size === 0) {
    return sparql_result;
  }

  if (!sparql_result.head.vars.includes("mane")) {
    sparql_result.head.vars.push("mane");
  }

  for (const binding of sparql_result.results.bindings) {
    const transcriptId = binding.enst_id?.value || String(binding.hgvs_c?.value || "").split(":")[0];

    if (maneSelectTranscripts.has(transcriptId)) {
      binding.mane = {
        type: "literal",
        value: "MANE_Select"
      };
    }
  }

  return sparql_result;
}
```
