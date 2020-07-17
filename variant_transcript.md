# Variant report / Transcript

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv219804
* `assembly` assembly
  * default: GRCh37
  * example: GRCh37, GRCh38

## Endpoint

{{ep}}

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

PREFIX dct: <http://purl.org/dc/terms/>
PREFIX dc11: <http://purl.org/dc/elements/1.1/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX m2r: <http://med2rdf.org/ontology/med2rdf#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?transcript ?enst_id ?gene_symbol ?gene_xref (GROUP_CONCAT(DISTINCT ?_consequence_label ; separator = ",") AS ?consequence_label) ?hgvs_p ?hgvs_c ?sift ?polyphen
FROM <http://togovar.biosciencedbc.jp/variation>
FROM <http://togovar.biosciencedbc.jp/so>
FROM <http://togovar.biosciencedbc.jp/{{graph}}>
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  ?variation dct:identifier ?tgv_id ;
             tgvo:hasConsequence ?_consequence .

  ?_consequence rdf:type ?_consequence_type .
  ?_consequence_type rdfs:label ?_consequence_label .

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
    OPTIONAL { ?transcript dct:identifier|dc11:identifier ?enst_id . }
  }
  OPTIONAL {
    ?_consequence tgvo:gene ?_gene .
    FILTER STRSTARTS(STR(?_gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")

    OPTIONAL {
      ?_gene rdfs:label ?gene_symbol .
    }

    OPTIONAL {
      ?_gene rdfs:seeAlso/^rdfs:seeAlso ?gene_xref .
      FILTER STRSTARTS(STR(?gene_xref), "http://identifiers.org/hgnc/")
    }
  }
}
ORDER BY ?transcript
```
