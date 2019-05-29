# Variant report / Frequency

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv18085

## Endpoint

{{{ep}}}

## `result` fetch basic information

```sparql
DEFINE sql:select-option "order"

PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX sio:  <http://semanticscience.org/resource/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>

SELECT ?source ?num_alleles ?num_ref_alleles ?num_alt_alleles ?num_genotype_ref_homo ?num_genotype_hetero ?num_genotype_alt_homo ?frequency ?filter ?quality
FROM <http://togovar.biosciencedbc.jp/graph/variant>
FROM <http://togovar.biosciencedbc.jp/graph/variant/frequency/tommo>
FROM <http://togovar.biosciencedbc.jp/graph/variant/frequency/jga_ngs>
FROM <http://togovar.biosciencedbc.jp/graph/variant/frequency/jga_snp>
FROM <http://togovar.biosciencedbc.jp/graph/variant/frequency/hgvd>
FROM <http://togovar.biosciencedbc.jp/graph/variant/frequency/exac>
WHERE {
  VALUES ?variant { <http://togovar.biosciencedbc.jp/variant/{{tgv_id}}> }

  {
    ?variant tgvo:hasFrequency ?_f .
    ?_f rdfs:label ?source ;
      tgvo:numAlleles ?num_alleles ;
      tgvo:numRefAlleles ?num_ref_alleles ;
      tgvo:numAltAlleles ?num_alt_alleles ;
      tgvo:frequency ?frequency .

    OPTIONAL { ?_f tgvo:numGenotypeRefHomo ?num_genotype_ref_homo . }
    OPTIONAL { ?_f tgvo:numGenotypeHetero ?num_genotype_hetero . }
    OPTIONAL { ?_f tgvo:numGenotypeAltHomo ?num_genotype_alt_homo . }

    OPTIONAL { ?_f tgvo:filter ?filter . }
    OPTIONAL { ?_f tgvo:quality ?quality . }
  } UNION {
    ?variant tgvo:hasFrequency ?_f .
    ?_f rdfs:label ?_source ;
      sio:SIO_000028 ?_population .
    ?_population rdfs:label ?_label ;
      tgvo:numAlleles ?num_alleles ;
      tgvo:numRefAlleles ?num_ref_alleles ;
      tgvo:numAltAlleles ?num_alt_alleles ;
      tgvo:frequency ?frequency .

    OPTIONAL { ?_population tgvo:numGenotypeRefHomo ?num_genotype_ref_homo . }
    OPTIONAL { ?_population tgvo:numGenotypeHetero ?num_genotype_hetero . }
    OPTIONAL { ?_population tgvo:numGenotypeAltHomo ?num_genotype_alt_homo . }

    OPTIONAL { ?_population tgvo:filter ?filter . }
    OPTIONAL { ?_population tgvo:quality ?quality . }

    BIND( CONCAT(?_source, ":", ?_label) AS ?source )
  }
}
```
