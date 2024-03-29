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

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  VALUES ?g {
    <http://togovar.org/variant/frequency/gem_j_wga>
    <http://togovar.org/variant/frequency/hgvd>
    <http://togovar.org/variant/frequency/jga_ngs>
    <http://togovar.org/variant/frequency/jga_snp>
    <http://togovar.org/variant/frequency/tommo>
    <http://togovar.org/variant/frequency/gnomad_genomes>
    <http://togovar.org/variant/frequency/gnomad_exomes>
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
    tommo: "ToMMo",
    hgvd: "HGVD",
    gnomad_genomes: "gnomAD Genomes",
    gnomad_genomes_afr: "gnomAD Genomes:African-American/African",
    gnomad_genomes_ami: "gnomAD Genomes:Amish",
    gnomad_genomes_amr: "gnomAD Genomes:Latino",
    gnomad_genomes_asj: "gnomAD Genomes:Ashkenazi Jewish",
    gnomad_genomes_eas: "gnomAD Genomes:East Asian",
    gnomad_genomes_fin: "gnomAD Genomes:Finnish",
    gnomad_genomes_mid: "gnomAD Genomes:Middle Eastern",
    gnomad_genomes_nfe: "gnomAD Genomes:Non-Finnish European",
    gnomad_genomes_oth: "gnomAD Genomes:Other",
    gnomad_genomes_remaining: "gnomAD Genomes:Remaining",
    gnomad_genomes_sas: "gnomAD Genomes:South Asian",
    gnomad_exomes: "gnomAD Exomes",
    gnomad_exomes_afr: "gnomAD Exomes:African-American/African",
    gnomad_exomes_amr: "gnomAD Exomes:Latino",
    gnomad_exomes_asj: "gnomAD Exomes:Ashkenazi Jewish",
    gnomad_exomes_eas: "gnomAD Exomes:East Asian",
    gnomad_exomes_fin: "gnomAD Exomes:Finnish",
    gnomad_exomes_mid: "gnomAD Exomes:Middle Eastern",
    gnomad_exomes_nfe: "gnomAD Exomes:Non-Finnish European",
    gnomad_exomes_oth: "gnomAD Exomes:Other",
    gnomad_exomes_remaining: "gnomAD Exomes:Remaining",
    gnomad_exomes_sas: "gnomAD Exomes:South Asian",
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
    ["gnomad_genomes", "_ami"],
    ["gnomad_genomes", "_amr"],
    ["gnomad_genomes", "_asj"],
    ["gnomad_genomes", "_eas"],
    ["gnomad_genomes", "_fin"],
    ["gnomad_genomes", "_mid"],
    ["gnomad_genomes", "_nfe"],
    ["gnomad_genomes", "_oth"],
    ["gnomad_genomes", "_remaining"],
    ["gnomad_genomes", "_sas"],
    ["gnomad_exomes", ""],
    ["gnomad_exomes", "_afr"],
    ["gnomad_exomes", "_amr"],
    ["gnomad_exomes", "_asj"],
    ["gnomad_exomes", "_eas"],
    ["gnomad_exomes", "_fin"],
    ["gnomad_exomes", "_mid"],
    ["gnomad_exomes", "_nfe"],
    ["gnomad_exomes", "_oth"],
    ["gnomad_exomes", "_remaining"],
    ["gnomad_exomes", "_sas"],
  ];

  return items.map(([name, suffix]) => {
    const hash = bindings.filter(x => x.source == name)
    
    const ac = hash.find(x => x.info_label == "AC" + suffix)?.info_value;
    const an = hash.find(x => x.info_label == "AN" + suffix)?.info_value;
    let af = hash.find(x => x.info_label == "AF" + suffix)?.info_value;
    if (!af) {
      if (ac && an) {
        af = ac / an;
      } else {
        af = 0
      }
    }

    return {
      source: display_source[name + suffix],
      num_alt_alleles: ac,
      num_alleles: an,
      frequency: af,
      num_genotype_ref_homo: hash.find(x => x.info_label == "RRC")?.info_value,
      num_genotype_hetero: hash.find(x => x.info_label == "ARC")?.info_value,
      num_genotype_alt_homo: hash.find(x => x.info_label == "AAC")?.info_value,
      filter: suffix.length == 0 ? hash[0]?.filter : "",
      quality: suffix.length == 0 ? hash[0]?.quality : "",
    };
  }).filter(x => x.frequency);
};
```
