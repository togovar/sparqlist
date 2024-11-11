# Disease report / Clinvar

## Parameters

* `medgen_cid` MedGen CID
  * default: C0023467
  * example: C0678222

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `medgen_clinvar`

```sparql
DEFINE sql:select-option "order"

PREFIX cvo:    <http://purl.jp/bio/10/clinvar/>
PREFIX dct:    <http://purl.org/dc/terms/>
PREFIX medgen: <http://ncbi.nlm.nih.gov/medgen/>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>
PREFIX sio:    <http://semanticscience.org/resource/>
PREFIX tgvo:   <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?rs_id ?variant ?title ?interpretation ?review_status ?last_evaluated ?condition
WHERE {
  VALUES ?medgen { medgen:{{medgen_cid}} }

  GRAPH <http://togovar.org/clinvar> {
    ?medgen ^dct:references ?_classified_condition .

    ?_classified_condition ^cvo:classified_condition/^cvo:classified_condition_list ?_rcv ;
      rdfs:label ?condition .

    ?_rcv cvo:rcv_classifications/cvo:germline_classification/cvo:description/cvo:description ?interpretation ;
      cvo:rcv_classifications/cvo:germline_classification/cvo:description/cvo:date_last_evaluated ?last_evaluated ;
      ^cvo:rcv_accession/^cvo:rcv_list/^cvo:classified_record ?clinvar .
      
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:variation_id ?vid ;
      cvo:classified_record/cvo:classifications/cvo:germline_classification/cvo:review_status ?review_status ;
      cvo:classified_record/sio:SIO_000628/dct:references ?dbsnp .

    BIND(STR(?vid) AS ?variation_id)

    ?dbsnp rdfs:seeAlso ?rs_id ;
      dct:source ?dbname .
    FILTER(?dbname IN ("dbSNP"))
  }

  GRAPH <http://togovar.org/variant/annotation/clinvar> {
    ?variant dct:identifier ?variation_id .
  }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```

## `result`

```javascript
({medgen_clinvar}) => {
  const clinical_significance_key = (interpretation) => {
    switch (interpretation.toLowerCase()) {
      case "pathogenic":
        return "P";
      case "likely pathogenic":
        return "LP";
      case "uncertain significance":
        return "US";
      case "likely benign":
        return "LB";
      case "benign":
        return "B";
      case "conflicting interpretations of pathogenicity":
        return "CI";
      case "drug response":
        return "DR";
      case "association":
        return "A";
      case "risk factor":
        return "RF";
      case "protective":
        return "PR";
      case "affects":
        return "AF";
      case "other":
        return "O";
      case "not provided":
        return "NP";
      case "association_not found":
        return "AN";
      default:
    }
  };

  const review_status_stars = (review_status) => {
    switch (review_status) {
      case "no assertion provided":
        return 0;
      case "no assertion criteria provided":
        return 0;
      case "no assertion for the individual variant":
        return 0;
      case "criteria provided, single submitter":
        return 1;
      case "criteria provided, conflicting interpretations":
        return 1;
      case "criteria provided, multiple submitters, no conflicts":
        return 2;
      case "reviewed by expert panel":
        return 3;
      case "practice guideline":
        return 4;
      default:
    }
  };

  return medgen_clinvar.results.bindings.map(x => {
    const position = x.variant.value.match(/http:\/\/identifiers.org\/hco\/(.+)\/GRCh3[78]#(\d+)/);

    return {
      tgv_id: x.tgv_id.value,
      tgv_link: "/variant/" + x.tgv_id.value,
      rs_id: x.rs_id.value.replace("http://ncbi.nlm.nih.gov/snp/", ""),
      rs_id_link: x.rs_id.value.replace("http://", "https://"),
      position: position[1] + ":" + position[2],
      title: x.title.value,
      interpretation: `<span class="clinical-significance-full" data-sign="${clinical_significance_key(x.interpretation.value)}">${x.interpretation.value}</span>`,
      review_status: `<span class="star-rating"><span data-stars="${review_status_stars(x.review_status.value)}" class="star-rating-item"></span></span><br><span class="status-description">${x.review_status.value}</span>`,
      last_evaluated: x.last_evaluated.value,
    };
  });
}
```
