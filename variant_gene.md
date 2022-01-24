# Variant report / Gene

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
#DEFINE sql:select-option "order"

PREFIX dct: <http://purl.org/dc/terms/>
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?variation ?gene ?hgnc ?symbol ?approved_name ?synonym
FROM <http://togovar.biosciencedbc.jp/variation>
FROM <http://togovar.biosciencedbc.jp/{{graph}}>
FROM <http://togovar.biosciencedbc.jp/hgnc>
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  ?variation dct:identifier ?tgv_id ;
             tgvo:hasConsequence ?_cn .

  ?_cn tgvo:gene ?gene .
  FILTER STRSTARTS(STR(?gene), "http://rdf.ebi.ac.uk/resource/ensembl/ENSG")

  OPTIONAL {
    ?gene rdfs:label ?symbol .
  }
  OPTIONAL {
    ?gene rdfs:seeAlso/^rdfs:seeAlso ?hgnc .
    FILTER STRSTARTS(STR(?hgnc), "http://identifiers.org/hgnc/")
    OPTIONAL { ?hgnc dct:description ?approved_name . }
    OPTIONAL { ?hgnc skos:altLabel ?synonym . }
  }
}
```

## `result`
#```javascript
#({variant_gene})=>{
#  return variant_gene.results.bindings.map(d=>{
#    return {
#      variation: d.variation.value,
#      gene: d.gene.value,
#      symbol: d.symbol.value,
#      approved_name: d.approved_name.value,
#      synomym: d.synonym.value
#    };
#  });
#}
#```
