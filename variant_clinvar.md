# Variant report / ClinVar

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

## `result`

```sparql
PREFIX cvo:   <http://purl.jp/bio/10/clinvar/>
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX gvo:   <http://genome-variation.org/resource#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?title ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv
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
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/clinvar> {
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
        dct:identifier ?variation_id .

      BIND(IRI(CONCAT("http://ncbi.nlm.nih.gov/clinvar/variation/", ?variation_id)) AS ?clinvar)
  }

  GRAPH <http://togovar.biosciencedbc.jp/clinvar> {
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?_rcv cvo:interpretation ?interpretation ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .

    ?_interpreted_condition rdfs:label ?condition .

    OPTIONAL {
      ?_interpreted_condition dct:source ?db ;
      dct:identifier ?medgen .
      FILTER (?db IN ("MedGen"))
    }
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```
