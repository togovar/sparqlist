# Get citation count from Colil

## Parameters

* `pmids` comma separated list of PubMed IDs
  * example: 21521872,28492532


## Endpoint

https://colil.dbcls.jp/sparql


## `array_of_pmids` Compile results

```javascript
({pmids}) => {
  return pmids.split(",");
}
```

## `sparql`

```sparql
DEFINE sql:select-option "order"

PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX colil:  <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{#each array_of_pmids}}'{{this}}' {{/each}} }

  GRAPH <http://purl.jp/bio/10/colil/core> {
    ?pmid ^togows:pmid ?pubmed .
    ?pubmed a colil:PubMed ;
      ^rdfs:seeAlso/^bibo:cites ?citation_paper.
  }
}
```

## `result` Compile results

```javascript
({ sparql }) => {
  return sparql.results.bindings.reduce((acc, x) => {
    acc[Number(x.pmid.value)] = Number(x.citation_count.value);
    return acc;
  }, {});
}
```
