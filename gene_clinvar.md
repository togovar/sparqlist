# Gene report / Clinvar

## Parameters

* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `hgnc_id` HGNC Id
  * default: 404

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

## `result`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX dct: <http://purl.org/dc/terms/>

SELECT DISTINCT ?tgv_id ?review_status ?interpretation ?last_evaluated ?condition ?medgen ?clinvar ?vcv ?vcv_disp

WHERE {
  VALUES ?ens_gene { <{{ ensg_uri_string }}> }

  GRAPH <http://togovar.biosciencedbc.jp/variation>{
    ?togovar tgvo:hasConsequence / tgvo:gene ?ens_gene .
    ?togovar dct:identifier ?tgv_id .
  }
  GRAPH <http://togovar.biosciencedbc.jp/variation/annotation/clinvar>{
    ?togovar tgvo:condition / rdfs:seeAlso ?clinvar .
  }

  GRAPH <http://togovar.biosciencedbc.jp/clinvar>{
    ?clinvar a cvo:VariationArchiveType ;
      rdfs:label ?title ;
      cvo:accession ?vcv ;
      cvo:interpreted_record/cvo:review_status ?review_status ;
      cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .

    BIND (REPLACE (STR(?vcv), 'VCV0+', '') AS ?vcv_disp) .

    ?_rcv cvo:interpretation ?interpretation ;
      dct:identifier ?rcv ;
      cvo:date_last_evaluated ?last_evaluated ;
      cvo:interpreted_condition_list/cvo:interpreted_condition ?_interpreted_condition .

    OPTIONAL {
      ?_interpreted_condition dct:references ?medgen ;
        dct:source ?db ;
        rdfs:label ?condition .
        FILTER (?db IN ("MedGen")) .
    }
  }
}
ORDER BY ?title ?review_status ?interpretation DESC(?last_evaluated) ?condition
```