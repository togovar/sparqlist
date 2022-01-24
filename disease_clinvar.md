# Disease report / Clinvar

## Parameters

* `medgen_cid` MedGen CID 
  * default: C0023467
  * example: C0678222

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `medgen_clinvar`

```sparql
PREFIX cvo: <http://purl.jp/bio/10/clinvar/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX medgen: <http://ncbi.nlm.nih.gov/medgen/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX sio: <http://semanticscience.org/resource/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?tgv_label ?rs_id ?title ?vcv_disp ?clinvar ?interpretation ?review_status ?last_evaluated ?condition
FROM <http://togovar.biosciencedbc.jp/clinvar>
FROM <http://togovar.biosciencedbc.jp/variant/annotation/clinvar>
FROM <http://togovar.biosciencedbc.jp/variant>
WHERE {
  VALUES ?medgen { medgen:{{ medgen_cid }} }

  GRAPH <http://togovar.biosciencedbc.jp/clinvar>{
    ?_interpreted_condition dct:references ?medgen;
        rdfs:label ?condition .

    ?_rcv cvo:interpretation ?interpretation ;
      dct:identifier ?rcv ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .

    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/sio:SIO_000628/dct:references ?dbsnp ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv.

    ?dbsnp rdfs:seeAlso ?rs_id;
      dct:source ?dbname.
    FILTER(?dbname IN ("dbSNP")).

    BIND (REPLACE (STR(?vcv), 'VCV0+', '') AS ?vcv_disp)
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/clinvar>{
    ?togovar tgvo:condition / rdfs:seeAlso ?clinvar .
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant>{
    ?togovar dct:identifier ?tgv_id .
    ?togovar rdfs:label ?tgv_label
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```

## `clinical_significance_key`

```javascript
({medgen_clinvar}) => {
  let ref = {};
  let key;
  medgen_clinvar.results.bindings.forEach((x) => {
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
({medgen_clinvar}) => {
  let ref = {};
  let stars;
  medgen_clinvar.results.bindings.forEach((x) => {
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
({medgen_clinvar, clinical_significance_key, review_status_stars}) => {
  return medgen_clinvar.results.bindings.map(d => ({
    tgv_id: d.tgv_id.value,
    tgv_link: "/variant/" + d.tgv_id.value,
    rs_id: d.rs_id.value.replace("http://ncbi.nlm.nih.gov/snp/", ""),
    rs_id_link: d.rs_id.value.replace("http://", "https://"),
    position: d.tgv_label.value.split("-")[0] + ":" + d.tgv_label.value.split("-")[1],
    title: d.title.value,
    vcv: d.vcv_disp.value,
    clinvar: d.clinvar.value,
    interpretation: clinical_significance_key[d.interpretation.value],
    review_status: review_status_stars[d.review_status.value],
    last_evaluated: d.last_evaluated.value
  }));
}
```
