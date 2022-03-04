# Variant report / Frequency

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `type`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>

SELECT ?type
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variation dct:identifier ?tgv_id ;
      a ?_type .
    BIND(REPLACE(STR(?_type), "http://genome-variation.org/resource#", "") AS ?type)
  }
}
LIMIT 1
```

## `type`

```javascript
({type}) => {
  const t = type.results.bindings[0].type.value
  return { 
    snv: t == "SNV",
    ins: t == "Insertion",
    del: t == "Deletion",
    indel: t == "Indel",
    mnv: t == "MNV",
  };
}
```

## `info`

```sparql
{{#unless type.snv}}
DEFINE sql:select-option "order"
{{/unless}}

PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX rdf:   <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?source (GROUP_CONCAT(DISTINCT ?_filter ; separator = ", ") AS ?filter) ?quality ?info_label ?info_value
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variation dct:identifier ?tgv_id ;
      gvo:ref ?ref ;
      gvo:alt ?alt ;
      faldo:location ?location .

  {{#if type.snv}}
    ?location a faldo:ExactPosition ;
      faldo:position ?pos1 ;
      faldo:reference ?reference .
  {{else if type.del}}
    ?location a faldo:Region ;
      faldo:begin ?begin ;
      faldo:end ?end .

    ?begin a faldo:InBetweenPosition ;
      faldo:after ?pos1 ;
      faldo:before ?pos2 ;
      faldo:reference ?reference .

    ?end a faldo:InBetweenPosition ;
      faldo:after ?pos3 ;
      faldo:before ?pos4 .
  {{else if type.indel}}
    ?location a faldo:Region ;
      faldo:begin ?begin ;
      faldo:end ?end .

    ?begin a faldo:InBetweenPosition ;
      faldo:after ?pos1 ;
      faldo:before ?pos2 ;
      faldo:reference ?reference .

    ?end a faldo:InBetweenPosition ;
      faldo:after ?pos3 ;
      faldo:before ?pos4 .
  {{else if type.ins}}
    ?location a faldo:InBetweenPosition ;
      faldo:after ?pos1 ;
      faldo:before ?pos2 ;
      faldo:reference ?reference .
  {{else if type.mnv}}
    ?location a faldo:Region ;
      faldo:begin ?pos1 ;
      faldo:end ?pos2 ;
      faldo:reference ?reference .
  {{/if}}

    BIND(STR(?variation) AS ?label)    
  }

  {
    VALUES ?g { <http://togovar.biosciencedbc.jp/variant/frequency/gem_j_wga> <http://togovar.biosciencedbc.jp/variant/frequency/hgvd> <http://togovar.biosciencedbc.jp/variant/frequency/jga_ngs> <http://togovar.biosciencedbc.jp/variant/frequency/jga_snp> <http://togovar.biosciencedbc.jp/variant/frequency/tommo_8.3kjpn> }
    GRAPH ?g {
      ?variation gvo:info ?info .

      ?info rdfs:label ?info_label ;
        rdf:value ?info_value .

      OPTIONAL { ?variation gvo:filter ?_filter . }
      OPTIONAL { ?variation gvo:qual ?quality . }

      BIND(REPLACE(STR(?g), ".*/", "") AS ?source)
    }
  } UNION {
    GRAPH <http://togovar.biosciencedbc.jp/variant/frequency/gnomad_genomes> {
    {{#if type.snv}}
      ?location2 a faldo:ExactPosition ;
        faldo:position ?pos1 ;
        faldo:reference ?reference .
    {{else if type.del}}
      ?location2 a faldo:Region ;
        faldo:begin ?begin ;
        faldo:end ?end .

      ?begin a faldo:InBetweenPosition ;
        faldo:after ?pos1 ;
        faldo:before ?pos2 ;
        faldo:reference ?reference .

      ?end a faldo:InBetweenPosition ;
        faldo:after ?pos3 ;
        faldo:before ?pos4 .
    {{else if type.indel}}
      ?location2 a faldo:Region ;
        faldo:begin ?begin ;
        faldo:end ?end .

      ?begin a faldo:InBetweenPosition ;
        faldo:after ?pos1 ;
        faldo:before ?pos2 ;
        faldo:reference ?reference .

      ?end a faldo:InBetweenPosition ;
        faldo:after ?pos3 ;
        faldo:before ?pos4 .
    {{else if type.ins}}
      ?location2 a faldo:InBetweenPosition ;
        faldo:after ?pos1 ;
        faldo:before ?pos2 ;
        faldo:reference ?reference .
    {{else if type.mnv}}
      ?location2 a faldo:Region ;
        faldo:begin ?pos1 ;
        faldo:end ?pos2 ;
        faldo:reference ?reference .
    {{/if}}

      ?s gvo:ref ?ref ;
        gvo:alt ?alt ;
        faldo:location ?location2 ;
        gvo:info ?info .

      ?info rdfs:label ?info_label ;
          rdf:value ?info_value .

      OPTIONAL { ?s gvo:filter ?_filter . }
      OPTIONAL { ?s gvo:qual ?quality . }

      BIND ("gnomad_genomes" AS ?source)
    }
  } UNION {
    GRAPH <http://togovar.biosciencedbc.jp/variant/frequency/gnomad_exomes> {
    {{#if type.snv}}
      ?location2 a faldo:ExactPosition ;
        faldo:position ?pos1 ;
        faldo:reference ?reference .
    {{else if type.del}}
      ?location2 a faldo:Region ;
        faldo:begin ?begin ;
        faldo:end ?end .

      ?begin a faldo:InBetweenPosition ;
        faldo:after ?pos1 ;
        faldo:before ?pos2 ;
        faldo:reference ?reference .

      ?end a faldo:InBetweenPosition ;
        faldo:after ?pos3 ;
        faldo:before ?pos4 .
    {{else if type.indel}}
      ?location2 a faldo:Region ;
        faldo:begin ?begin ;
        faldo:end ?end .

      ?begin a faldo:InBetweenPosition ;
        faldo:after ?pos1 ;
        faldo:before ?pos2 ;
        faldo:reference ?reference .

      ?end a faldo:InBetweenPosition ;
        faldo:after ?pos3 ;
        faldo:before ?pos4 .
    {{else if type.ins}}
      ?location2 a faldo:InBetweenPosition ;
        faldo:after ?pos1 ;
        faldo:before ?pos2 ;
        faldo:reference ?reference .
    {{else if type.mnv}}
      ?location2 a faldo:Region ;
        faldo:begin ?pos1 ;
        faldo:end ?pos2 ;
        faldo:reference ?reference .
    {{/if}}

      ?s gvo:ref ?ref ;
        gvo:alt ?alt ;
        faldo:location ?location2 ;
        gvo:info ?info .

      ?info rdfs:label ?info_label ;
          rdf:value ?info_value .

      OPTIONAL { ?s gvo:filter ?_filter . }
      OPTIONAL { ?s gvo:qual ?quality . }

      BIND ("gnomad_exomes" AS ?source)
    }
  }
}
ORDER BY ?source
```

## `result`

```javascript
({info}) => {
  const source = {
    "gem_j_wga": "GEM-J WGA",
    "jga_ngs": "JGA-NGS",
    "jga_snp": "JGA-SNP",
    "tommo_8.3kjpn": "ToMMo 8.3KJPN",
    "hgvd": "HGVD",
    "gnomad_genomes": "gnomAD Genomes",
    "gnomad_genomes_afr": "gnomAD Genomes:African-American/African",
    "gnomad_genomes_amr": "gnomAD Genomes:Latino",
    "gnomad_genomes_oth": "gnomAD Genomes:Other",
    "gnomad_genomes_nfe": "gnomAD Genomes:Non-Finnish European",
    "gnomad_genomes_asj": "gnomAD Genomes:Ashkenazi Jewish",
    "gnomad_genomes_eas": "gnomAD Genomes:East Asian",
    "gnomad_genomes_fin": "gnomAD Genomes:Finnish",
    "gnomad_exomes": "gnomAD Exomes",
    "gnomad_exomes_afr": "gnomAD Exomes:African-American/African",
    "gnomad_exomes_amr": "gnomAD Exomes:Latino",
    "gnomad_exomes_sas": "gnomAD Exomes:South Asian",
    "gnomad_exomes_oth": "gnomAD Exomes:Other",    
    "gnomad_exomes_nfe": "gnomAD Exomes:Non-Finnish European",
    "gnomad_exomes_asj": "gnomAD Exomes:Ashkenazi Jewish",
    "gnomad_exomes_eas": "gnomAD Exomes:East Asian",
    "gnomad_exomes_fin": "gnomAD Exomes:Finnish",
  }

  const bindings = info.results.bindings.map(binding => Object.keys(binding).reduce((hash, key) => ({ ...hash, [key]: binding[key].value}), {}));

  return [
    ["gem_j_wga", ""],
    ["jga_ngs", ""],
    ["jga_snp", ""],
    ["tommo_8.3kjpn", ""],
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
  ].map(([name, suffix]) => {
    const hash = bindings.filter(x => x.source == name)

    return {
      "source": source[name + suffix],
      "num_alleles": hash.find(x => x.info_label == "AN" + suffix)?.info_value,
      "num_alt_alleles": hash.find(x => x.info_label == "AC" + suffix)?.info_value,
      "frequency": hash.find(x => x.info_label == "AF" + suffix)?.info_value,
      "num_genotype_ref_homo": hash.find(x => x.info_label == "RRC")?.info_value,
      "num_genotype_hetero": hash.find(x => x.info_label == "ARC")?.info_value,
      "num_genotype_alt_homo": hash.find(x => x.info_label == "AAC")?.info_value,
      "filter": suffix.length == 0 ? hash[0]?.filter : "",
      "quality": suffix.length == 0 ? hash[0]?.quality : "",
    };
  }).filter(x => x.frequency);
};
```
