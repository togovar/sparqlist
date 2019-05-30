# Variant report / Transcript

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv2479309
* `assembly` assembly
  * default: GRCh38
  * example: GRCh37, GRCh38

## Endpoint

{{{ep}}}

## `graph`

```javascript
({
  json({assembly}) {
    return assembly === "GRCh38" ? "ensembl38" : "ensembl37" ;
  }
})
```

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX dc11: <http://purl.org/dc/elements/1.1/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX m2r: <http://med2rdf.org/ontology/med2rdf#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>

SELECT ?transcript ?enst_id ?gene_symbol ?gene_xref ?consequence_label ?hgvs_p ?hgvs_c ?sift ?polyphen
FROM <http://togovar.biosciencedbc.jp/graph/variant>
FROM <http://togovar.biosciencedbc.jp/graph/so>
FROM <http://togovar.biosciencedbc.jp/graph/{{graph}}>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }

  ?variant tgvo:hasConsequence ?_consequence .
  ?_consequence rdf:type ?_consequence_type .
  ?_consequence_type rdfs:label ?consequence_label .

  OPTIONAL { ?_consequence tgvo:sift ?sift . }
  OPTIONAL { ?_consequence tgvo:polyphen ?polyphen . }
  OPTIONAL {
    ?_consequence rdfs:label ?hgvs_p .
    FILTER REGEX(?hgvs_p, "^ENSP")
  }
  OPTIONAL {
    ?_consequence rdfs:label ?hgvs_c .
    FILTER REGEX(?hgvs_c, "^ENST")
  }

  OPTIONAL {
    ?_consequence tgvo:transcript ?transcript . 
    OPTIONAL { ?transcript dc11:identifier ?enst_id . }
  }
  OPTIONAL {
    ?_consequence tgvo:gene ?_gene .
    ?_gene rdfs:label ?gene_symbol .
    OPTIONAL {
      ?_gene rdfs:seeAlso ?gene_xref .
      FILTER REGEX(STR(?gene_xref), "^http://identifiers.org/hgnc/")
    }
  }
}
ORDER BY ?transcript
```
