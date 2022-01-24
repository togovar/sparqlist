# Variant report / Frequency

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{ep}}

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX dct: <http://purl.org/dc/terms/>
PREFIX sio: <http://semanticscience.org/resource/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX exo: <http://purl.jp/bio/10/exac/>

SELECT ?source ?num_alleles ?num_alt_alleles ?frequency ?num_genotype_ref_homo ?num_genotype_hetero ?num_genotype_alt_homo (GROUP_CONCAT(DISTINCT ?_filter ; separator = ", ") AS ?filter) ?quality
FROM <http://togovar.biosciencedbc.jp/variation>
FROM <http://togovar.biosciencedbc.jp/variation/frequency/exac>
FROM <http://togovar.biosciencedbc.jp/variation/frequency/gem_j_wga>
FROM <http://togovar.biosciencedbc.jp/variation/frequency/hgvd>
FROM <http://togovar.biosciencedbc.jp/variation/frequency/jga_ngs>
FROM <http://togovar.biosciencedbc.jp/variation/frequency/jga_snp>
FROM <http://togovar.biosciencedbc.jp/variation/frequency/tommo_4.7kjpn>
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  ?variation dct:identifier ?tgv_id ;
             rdfs:label ?label .

  {
    ?variation tgvo:statistics ?_stat .

    ?_stat dct:source ?source ;
        tgvo:alleleNumber ?num_alleles ;
        tgvo:alleleCount ?num_alt_alleles ;
        tgvo:alleleFrequency ?frequency .

    OPTIONAL { ?_stat tgvo:filter ?_filter . }
    OPTIONAL { ?_stat tgvo:quality ?_quality . }

    BIND (IF(?source IN ("ToMMo 4.7KJPN", "HGVD", "JGA-SNP"), undef, ?_quality) AS ?quality)

    OPTIONAL { ?_stat tgvo:homozygousReferenceAlleleCount ?num_genotype_ref_homo . }
    OPTIONAL { ?_stat tgvo:heterozygousAlleleCount ?num_genotype_hetero . }
    OPTIONAL { ?_stat tgvo:homozygousAlternativeAlleleCount ?num_genotype_alt_homo . }
  } UNION {
    ?exac dct:identifier ?label ;
          exo:alleleCount ?num_alt_alleles ;
          exo:alleleNum ?num_alleles ;
          exo:alleleFrequency ?frequency .

    OPTIONAL { ?exac exo:filter ?_filter . }
    OPTIONAL { ?exac tgvo:statistics/tgvo:quality ?quality . }

    BIND ("ExAC" AS ?source)
  } UNION {
    ?exac dct:identifier ?label ;
          exo:population ?_pop .

    ?_pop a exo:Population ;
            rdfs:label ?_population ;
            exo:alleleCount ?num_alt_alleles ;
            exo:alleleNum ?num_alleles ;
            exo:alleleFrequency ?frequency .

    BIND (CONCAT("ExAC:", ?_population) AS ?source)
  }
}
ORDER BY ?source
```
