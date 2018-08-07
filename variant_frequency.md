# TogoVar variant report variant frequency

Obtain variant frequency

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 30913364

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `result`

```sparql
PREFIX tgv: <http://togovar.org/lookup#>

SELECT DISTINCT *
FROM <http://togovar.org/graph/lookup>
WHERE {
  VALUES ?var { <http://togovar.org/variation/{{tgv_id}}> }
  {
    ?var tgv:jga_ngs/tgv:num_alleles ?num_all ;
      tgv:jga_ngs/tgv:num_alt_alleles ?num_alt ;
      tgv:jga_ngs/tgv:frequency ?freq ;
      tgv:jga_ngs/tgv:passed ?pass .
    BIND("JGA-NGS" AS ?dataset)
    BIND("Japanese" AS ?population)
  } UNION {
    ?var tgv:jga_snp/tgv:num_alleles ?num_all ;
      tgv:jga_snp/tgv:num_alt_alleles ?num_alt ;
      tgv:jga_snp/tgv:frequency ?freq ;
      tgv:jga_snp/tgv:num_genotype_ref_homo ?num_genotype_ref_homo ;
      tgv:jga_snp/tgv:num_genotype_hetero ?num_genotype_hetero ;
      tgv:jga_snp/tgv:num_genotype_alt_homo ?num_genotype_alt_homo .
    BIND("JGA-SNP" AS ?dataset)
    BIND("Japanese" AS ?population)
  } UNION {
    ?var tgv:tommo/tgv:num_alleles ?num_all ;
      tgv:tommo/tgv:num_alt_alleles ?num_alt ;
      tgv:tommo/tgv:frequency ?freq ;
      tgv:tommo/tgv:passed ?pass .
    BIND("3.5 KJPN" AS ?dataset)
    BIND("Japanese" AS ?population)
  } UNION {
    ?var tgv:hgvd/tgv:num_alleles ?num_all ;
      tgv:hgvd/tgv:num_alt_alleles ?num_alt ;
      tgv:hgvd/tgv:frequency ?freq ;
      tgv:hgvd/tgv:passed ?pass .
    BIND("HGVD" AS ?dataset)
    BIND("Japanese" AS ?population)
  } UNION {
    ?var tgv:exac/tgv:num_alleles ?num_all ;
      tgv:exac/tgv:num_alt_alleles ?num_alt ;
      tgv:exac/tgv:frequency ?freq ;
      tgv:exac/tgv:passed ?pass .
    BIND("ExAC" AS ?dataset)
    BIND("Total" AS ?population)
  }
}
```