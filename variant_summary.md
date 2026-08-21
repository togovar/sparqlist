# Variant report / Summary

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv56616325
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 16-89196249-G-A

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
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX tgvo:  <http://togovar.org/vocabulary/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos:  <http://www.w3.org/2004/02/skos/core#>

SELECT DISTINCT ?type ?reference ?position ?ref ?alt ?gene ?hgnc ?symbol ?approved_name
WHERE {
  VALUES ?variant { <{{variant}}> }

  GRAPH <http://togovar.org/variant> {
    ?variant a ?class ;
      faldo:location/faldo:begin?/faldo:reference ?reference ;
      gvo:pos_vcf ?position ;
      gvo:ref_vcf ?ref ;
      gvo:alt_vcf ?alt .

    BIND (
      IF(?class = gvo:SNV, "SNV", 
        IF(?class = gvo:Insertion, "Insertion", 
          IF(?class = gvo:Deletion, "Deletion", 
            IF(?class = gvo:Indel, "Indel", 
              IF(?class = gvo:MNV, "Substitution", "Unknown")
            )
          )
        )
      ) AS ?type
    )
  }

  OPTIONAL {
    GRAPH <http://togovar.org/variant/annotation/ensembl> {
      ?variant tgvo:hasConsequence/tgvo:gene ?gene ;
        tgvo:hasConsequence/tgvo:hgnc ?hgnc .
      FILTER STRSTARTS(STR(?gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")
    }

    GRAPH <http://togovar.org/hgnc> {
      ?hgnc rdfs:label ?symbol ;
        dct:description ?approved_name .
    }
  }
}
```

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_APP, SPARQLIST_TOGOVAR_REFERENCE, SPARQLIST_TOGOVAR_SPARQL, variant, sparql_result}) => {
  if (sparql_result.results.bindings.length > 0) {
    return sparql_result;
  }

  const regex = /^http:\/\/identifiers\.org\/hco\/(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)\/(?<reference>GRCh3[78])#(?<pos>\d+)-(?<ref>.+)-(?<alt>.+)$/;
  const match = variant.match(regex);

  if (!match || !match.groups) {
    return sparql_result;
  }

  if (match.groups.reference !== SPARQLIST_TOGOVAR_REFERENCE) {
    return sparql_result;
  }

  const position = parseInt(match.groups.pos, 10);
  const matchesVariant = x => {
    return String(x.chromosome) === match.groups.chr &&
      Number(x.position) === position &&
      x.reference === match.groups.ref &&
      x.alternate === match.groups.alt;
  };

  const searchVariant = async query => {
    try {
      const res = await fetch(SPARQLIST_TOGOVAR_APP.concat("/api/search/variant"), {
        method: "POST",
        headers: {
          "Accept": "application/json",
          "Content-Type": "application/json"
        },
        body: JSON.stringify({
          query,
          offset: 0,
          limit: 20
        })
      });

      if (!res.ok) {
        return [];
      }

      const json = await res.json();
      return Array.isArray(json.data) ? json.data : [];
    } catch (error) {
      return [];
    }
  };

  let data = (await searchVariant({
    variant: {
      chromosome: match.groups.chr,
      position: position,
      reference: match.groups.ref,
      alternate: match.groups.alt
    }
  })).find(matchesVariant);

  if (!data) {
    data = (await searchVariant({
      location: {
        chromosome: match.groups.chr,
        position: position
      }
    })).find(matchesVariant);
  }

  if (!data) {
    return sparql_result;
  }

  const type = {
    SO_0001483: "SNV",
    SO_0000667: "Insertion",
    SO_0000159: "Deletion",
    SO_1000032: "Indel",
    SO_0002007: "Substitution"
  }[data.type] || "Unknown";
  const gene = data.genes?.[0];
  const binding = {
    type: {type: "literal", value: type},
    reference: {type: "uri", value: `http://identifiers.org/hco/${data.chromosome}/${match.groups.reference}`},
    position: {type: "literal", datatype: "http://www.w3.org/2001/XMLSchema#integer", value: String(data.position)},
    ref: {type: "literal", value: data.reference},
    alt: {type: "literal", value: data.alternate}
  };

  if (String(gene?.id || "").match(/^\d+$/)) {
    binding.hgnc = {type: "uri", value: `http://identifiers.org/hgnc/${gene.id}`};
  }

  if (gene?.name) {
    binding.symbol = {type: "literal", value: gene.name};
  }

  if (String(gene?.id || "").match(/^\d+$/)) {
    const query = `
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?approved_name
WHERE {
  GRAPH <http://togovar.org/hgnc> {
    <http://identifiers.org/hgnc/${gene.id}> dct:description ?approved_name .
  }
}
LIMIT 1`;
    try {
      const res = await fetch(SPARQLIST_TOGOVAR_SPARQL.concat("?query=", encodeURIComponent(query), "&format=application%2Fsparql-results%2Bjson"));

      if (res.ok) {
        const json = await res.json();
        const approvedName = json.results.bindings[0]?.approved_name?.value;

        if (approvedName) {
          binding.approved_name = {type: "literal", value: approvedName};
        }
      }
    } catch (error) {
    }
  }

  return {
    head: sparql_result.head,
    results: {
      bindings: [binding]
    }
  };
}
```
