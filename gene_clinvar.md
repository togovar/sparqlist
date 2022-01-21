# Gene report / Clinvar

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `hgnc_id` HGNC Id
  * default: 404
* `base_url` TogoVar URL
  * default: https://togovar.biosciencedbc.jp

## Endpoint

{{ ep }}

## `ensg_uri_json`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?ens_gene_togovar
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc>{
    ?hgnc_uri rdfs:seeAlso ?ens_gene_hgnc .
    FILTER(CONTAINS(STR(?ens_gene_hgnc), "http://identifiers.org/ensembl/")) .
    BIND(IRI(REPLACE(STR(?ens_gene_hgnc), "http://identifiers.org/ensembl/","http://rdf.ebi.ac.uk/resource/ensembl/")) As ?ens_gene_togovar) .
  }
}
```

## `ensg_uri_string`

```javascript
({ensg_uri_json}) => {
  return ensg_uri_json.results.bindings[0].ens_gene_togovar.value;
}
```

## `ensg2clinvar`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX dct: <http://purl.org/dc/terms/>

SELECT DISTINCT ?tgv_id ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?title ?vcv ?label

WHERE {
  VALUES ?ens_gene { <{{ ensg_uri_string }}> }

  GRAPH <http://togovar.biosciencedbc.jp/variant>{
    ?togovar tgvo:hasConsequence / tgvo:gene ?ens_gene .
    ?togovar dct:identifier ?tgv_id .
    ?togovar rdfs:label ?label
  }
  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/clinvar>{
    ?togovar tgvo:condition / rdfs:seeAlso ?clinvar .
  }

  GRAPH <http://togovar.biosciencedbc.jp/clinvar>{
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?_rcv cvo:interpretation ?interpretation ;
      dct:identifier ?rcv ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .

    OPTIONAL {
      ?_interpreted_condition rdfs:label ?condition_label .
      ?_interpreted_condition dct:source ?db ;
                              dct:identifier ?medgen .
      FILTER (?db IN ("MedGen"))
    }

    BIND(IF(isLiteral(?_interpreted_condition), ?_interpreted_condition, ?condition_label) AS ?condition)
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```

## `clinical_significance_key`

```javascript
({ensg2clinvar}) => {
  let ref = {};
  let key;
  ensg2clinvar.results.bindings.forEach((x) => {
    switch (x.interpretation.value.toLowerCase()){
      case "pathogenic":
        key = "P";
        break;
      case "likely pathogenic":
        key = "LP";
        break;
      case "uncertain significance":
        key = "US";
        break;
      case "likely benign":
        key = "LB";
        break;
      case "benign":
        key = "B";
        break;
      case "conflicting interpretations of pathogenicity":
        key = "CI";
        break;
      case "drug response":
        key = "DR";
        break;
      case "association":
        key = "A";
        break;
      case "risk factor":
        key = "RF";
        break;
      case "protective":
        key = "PR";
        break;
      case "affects":
        key = "AF";
        break;
      case "other":
        key = "O";
        break;
      case "not provided":
        key = "NP";
        break;
      case "association_not found":
        key = "AN";
        break;
      default:
        break;
    }
    ref[x.interpretation.value] = '<span class="clinical-significance-full" data-sign="' +  key + '">' + x.interpretation.value + '</span>'
  });
  return ref
}
```

## `review_status_stars`

```javascript
({ensg2clinvar}) => {
  let ref = {};
  let stars;
  ensg2clinvar.results.bindings.forEach((x) => {
    switch (x.review_status.value){
      case "no assertion provided":
        stars = 0;
        break;
      case "no assertion criteria provided":
        stars = 0;
        break;
      case "no assertion for the individual variant":
        stars = 0;
        break;
      case "criteria provided, single submitter":
        stars = 1;
        break;
      case "criteria provided, conflicting interpretations":
        stars = 1;
        break;
      case "criteria provided, multiple submitters, no conflicts":
        stars = 2;
        break;
      case "reviewed by expert panel":
        stars = 3;
        break;
      case "practice guideline":
        stars = 4;
        break;
      default:
        break;
    }
    ref[x.review_status.value] = '<span class="star-rating">' + '<span data-stars="' + stars + '"' + 'class="star-rating-item">' + '</span></span><br>' + '<span class="status-description">' + x.review_status.value + '</span>';
  });
  return ref
}
```

## `result`

```javascript
({base_url, ensg2clinvar, clinical_significance_key, review_status_stars}) => {
  return ensg2clinvar.results.bindings.map(d => ({
    tgv_id: d.tgv_id.value,
    tgv_link: base_url + "/variant/" + d.tgv_id.value,
    position: d.label.value.split("-")[0] + ":" + d.label.value.split("-")[1],
    title: d.title.value,
    vcv: d.vcv.value.replace(/VCV0+/, ''),
    clinvar: d.clinvar.value,
    interpretation: clinical_significance_key[d.interpretation.value],
    review_status: review_status_stars[d.review_status.value],
    last_evaluated: d.last_evaluated.value,
    medgen: (d.medgen ? "https://www.ncbi.nlm.nih.gov/medgen/" + d.medgen.value : ""),
    condition: d.condition.value
  }));
}
```