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

## `rs2tgv`
```sparql
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX dct: <http://purl.org/dc/terms/>

SELECT ?dbsnp SAMPLE(?tgv_id) AS ?tgv_id
FROM <http://togovar.biosciencedbc.jp/variant>
WHERE {
 VALUES ?dbsnp { {{#each rs_ids}} dbsnp:{{ this }} {{/each}} } 
  ?variation dct:identifier ?tgv_id ;
    rdfs:seeAlso ?dbsnp.
}
```

## `result`
```javascript
({rs2tgv}) => {
  let result = {};
  rs2tgv.results.bindings.forEach(d => {
    result[d.dbsnp.value.replace("http://identifiers.org/dbsnp/","")] = d.tgv_id.value;
  });
  return result;
}