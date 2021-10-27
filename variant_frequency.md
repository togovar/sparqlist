# Variant report / Frequency

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv219804

## Endpoint

{{ep}}

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

## `result`

```sparql
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo: <http://genome-variation.org/resource#>

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
