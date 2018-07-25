# TogoVar ClinVar NanoStanza

Obtain variant significance of a TogoVar variant

## Parameters

* `tgv_id` TogoVar variant ID
  * default: 14198
  * examples: 37018, 36896

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `results`

```sparql
DEFINE sql:select-option "order"
PREFIX dc: <http://purl.org/dc/terms/>
PREFIX cvo: <http://purl.jp/bio/10/clinvar/>

SELECT DISTINCT ?significance
FROM <http://togovar.org/graph/tgclinvar>
FROM <http://togovar.org/graph/clinvar>
WHERE {
  VALUES ?tgv { <http://togovar.org/variation/{{tgv_id}}> }
  ?tgv rdfs:seeAlso ?allele .
  ?allele dc:identifier ?allele_id ;
    cvo:clinicalSignificance ?significance .
}
LIMIT 1
```

## `json`

```javascript
({
  json({results}) {
    if (result = results.results.bindings[0]) {
      return {
        "significance": result.significance.value
      };
    } else {
      return {};
    }
  }
})
```
