# Gene report / Clinvar

## Parameters

* `hgnc_id` HGNC Id
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `xref`

```sparql
PREFIX hgnc: <http://identifiers.org/hgnc/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?xref
WHERE {
  VALUES ?gene { hgnc:{{hgnc_id}} }

  GRAPH <http://togovar.org/hgnc> {
    ?gene rdfs:seeAlso ?xref .
    FILTER STRSTARTS(STR(?xref), "http://identifiers.org/ensembl/")
  }
}
```

## `ensembl_gene`

```javascript
({xref}) => {
  return xref.results.bindings.map(x => x.xref.value.replace("http://identifiers.org/ensembl/", ""));
}
```

## `ensg2clinvar`

```sparql
DEFINE sql:select-option "order"

PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX sio:  <http://semanticscience.org/resource/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?tgv_id ?rs_id ?variant ?title ?interpretation ?review_status ?last_evaluated ?condition ?medgen
WHERE {
  VALUES ?ens_gene { {{#each ensembl_gene}} ensg:{{this}} {{/each}} }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?ens_gene ^tgvo:gene/^tgvo:hasConsequence ?variant .
  }

  GRAPH <http://togovar.org/variant> {
    ?variant dct:identifier ?tgv_id .
  }
  
  GRAPH <http://togovar.org/variant/annotation/clinvar> {
    ?variant dct:identifier ?variation_id .

    BIND(IRI(CONCAT("http://ncbi.nlm.nih.gov/clinvar/variation/", ?variation_id)) AS ?clinvar)
  }

  GRAPH <http://togovar.org/clinvar> {
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/sio:SIO_000628/dct:references ?dbsnp ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    ?dbsnp rdfs:seeAlso ?rs_id ;
      dct:source ?dbname.
    FILTER(?dbname IN ("dbSNP"))

    ?_rcv cvo:interpretation ?interpretation ;
      dct:identifier ?rcv ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .

    OPTIONAL {
      ?_interpreted_condition rdfs:label ?condition .
    }
    OPTIONAL {
      ?_interpreted_condition dct:source ?db ;
        dct:identifier ?medgen .
      FILTER(?db IN ("MedGen"))
    }
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```

## `result`

```javascript
({ensg2clinvar}) => {
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

  return ensg2clinvar.results.bindings.map(x => {
    const position = x.variant?.value?.match(/http:\/\/identifiers.org\/hco\/(.+)\/GRCh3[78]#(\d+)/);

    return {
      tgv_id: x.tgv_id?.value,
      tgv_link: "/variant/" + x.tgv_id?.value,
      rs_id: x.rs_id?.value?.replace("http://ncbi.nlm.nih.gov/snp/", ""),
      rs_id_link: x.rs_id?.value?.replace("http://", "https://"),
      position: position ? position[1] + ":" + position[2] : null,
      title: x.title?.value,
      interpretation: `<span class="clinical-significance-full" data-sign="${clinical_significance_key(x.interpretation?.value)}">${x.interpretation?.value}</span>`,
      review_status: `<span class="star-rating"><span data-stars="${review_status_stars(x.review_status?.value)}" class="star-rating-item"></span></span><br><span class="status-description">${x.review_status?.value}</span>`,
      last_evaluated: x.last_evaluated?.value,
      condition: x.condition?.value,
      medgen: (x.medgen ? "https://www.ncbi.nlm.nih.gov/medgen/" + x.medgen.value : ""),
    };
  });
}
```
