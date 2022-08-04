# Variant report / Frequency

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `result`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX gvo:  <http://genome-variation.org/resource#>
PREFIX rdf:  <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?source (GROUP_CONCAT(DISTINCT ?_filter ; separator = ", ") AS ?filter) ?quality ?info_label ?info_value
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  VALUES ?g {
    <http://togovar.biosciencedbc.jp/variant/frequency/gem_j_wga>
    <http://togovar.biosciencedbc.jp/variant/frequency/hgvd>
    <http://togovar.biosciencedbc.jp/variant/frequency/jga_ngs>
    <http://togovar.biosciencedbc.jp/variant/frequency/jga_snp>
    <http://togovar.biosciencedbc.jp/variant/frequency/tommo>
    <http://togovar.biosciencedbc.jp/variant/frequency/gnomad_genomes>
    <http://togovar.biosciencedbc.jp/variant/frequency/gnomad_exomes>
  }

  GRAPH ?g {
    ?variant gvo:info [
      rdfs:label ?info_label ;
      rdf:value ?info_value
    ] .

    OPTIONAL { ?variant gvo:filter ?_filter . }
    OPTIONAL { ?variant gvo:qual ?quality . }

    BIND(REPLACE(STR(?g), ".*/", "") AS ?source)
  }
}
ORDER BY ?source
```

## `result`

```javascript
({result}) => {
  const display_source = {
    gem_j_wga: "GEM-J WGA",
    jga_ngs: "JGA-NGS",
    jga_snp: "JGA-SNP",
    tommo: "ToMMo 8.3KJPN",
    hgvd: "HGVD",
    gnomad_genomes: "gnomAD Genomes",
    gnomad_genomes_afr: "gnomAD Genomes:African-American/African",
    gnomad_genomes_amr: "gnomAD Genomes:Latino",
    gnomad_genomes_oth: "gnomAD Genomes:Other",
    gnomad_genomes_nfe: "gnomAD Genomes:Non-Finnish European",
    gnomad_genomes_asj: "gnomAD Genomes:Ashkenazi Jewish",
    gnomad_genomes_eas: "gnomAD Genomes:East Asian",
    gnomad_genomes_fin: "gnomAD Genomes:Finnish",
    gnomad_exomes: "gnomAD Exomes",
    gnomad_exomes_afr: "gnomAD Exomes:African-American/African",
    gnomad_exomes_amr: "gnomAD Exomes:Latino",
    gnomad_exomes_sas: "gnomAD Exomes:South Asian",
    gnomad_exomes_oth: "gnomAD Exomes:Other",
    gnomad_exomes_nfe: "gnomAD Exomes:Non-Finnish European",
    gnomad_exomes_asj: "gnomAD Exomes:Ashkenazi Jewish",
    gnomad_exomes_eas: "gnomAD Exomes:East Asian",
    gnomad_exomes_fin: "gnomAD Exomes:Finnish",
  }

  const bindings = result.results.bindings.map(binding => Object.keys(binding).reduce((hash, key) => ({
    ...hash,
    [key]: binding[key].value
  }), {}));

  const items = [
    ["gem_j_wga", ""],
    ["jga_ngs", ""],
    ["jga_snp", ""],
    ["tommo", ""],
    ["hgvd", ""],
    ["gnomad_genomes", ""],
    ["gnomad_genomes", "_afr"],
    ["gnomad_genomes", "_amr"],
    ["gnomad_genomes", "_oth"],
    ["gnomad_genomes", "_nfe"],
    ["gnomad_genomes", "_asj"],
    ["gnomad_genomes", "_eas"],
    ["gnomad_genomes", "_fin"],
    ["gnomad_exomes", ""],
    ["gnomad_exomes", "_afr"],
    ["gnomad_exomes", "_amr"],
    ["gnomad_exomes", "_sas"],
    ["gnomad_exomes", "_oth"],
    ["gnomad_exomes", "_nfe"],
    ["gnomad_exomes", "_asj"],
    ["gnomad_exomes", "_eas"],
    ["gnomad_exomes", "_fin"],
  ];

  return items.map(([name, suffix]) => {
    const hash = bindings.filter(x => x.source == name)

    return {
      source: display_source[name + suffix],
      num_alleles: hash.find(x => x.info_label == "AN" + suffix)?.info_value,
      num_alt_alleles: hash.find(x => x.info_label == "AC" + suffix)?.info_value,
      frequency: hash.find(x => x.info_label == "AF" + suffix)?.info_value,
      num_genotype_ref_homo: hash.find(x => x.info_label == "RRC")?.info_value,
      num_genotype_hetero: hash.find(x => x.info_label == "ARC")?.info_value,
      num_genotype_alt_homo: hash.find(x => x.info_label == "AAC")?.info_value,
      filter: suffix.length == 0 ? hash[0]?.filter : "",
      quality: suffix.length == 0 ? hash[0]?.quality : "",
    };
  }).filter(x => x.frequency);
};
```
