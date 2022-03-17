# Disease report / rs to TogoVarID for Disease-GWAS

## Parameters

* `rs_id` dbSNP
  * default: rs3809910,rs671

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `rs_ids`

```javascript
({rs_id}) => {
  return rs_id.split(',')
}
```

## `xref`

```sparql
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?dbsnp SAMPLE(?tgv_id) AS ?tgv_id
WHERE {
  VALUES ?dbsnp { {{#each rs_ids}} dbsnp:{{this}} {{/each}} }

  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/ensembl> {
    ?variant rdfs:seeAlso ?dbsnp .
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variant dct:identifier ?tgv_id .
  }
}
```

## `result`

```javascript
({xref}) => {
  const prefix = "http://identifiers.org/dbsnp/";

  let result = {};

  xref.results.bindings.forEach(x => {
    result[x.dbsnp.value.replace(prefix, "")] = x.tgv_id.value;
  });

  return result;
}
```
