# jBrowse Gene Track

## Parameters

* `query` {chromosome}:{start}:{end}:{assembly}
  * default: 12:112200000:112250000:GRCh37

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `q` Extract chromosome and start/end position

```javascript
({
  json({query}) {
    return query.match(/^([1-9]|1[0-9]|2[0-2]|X|Y|MT):(\d+):(\d+):(GRCh\d+)/).slice(1,5);
  }
})
```

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX obo:   <http://purl.obolibrary.org/obo/>
PREFIX so:    <http://purl.obolibrary.org/obo/so#>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?gene_id ?gene_type ?gene_start ?gene_end ?feat_id ?feat_type ?feat_start ?feat_end ?exon_id ?exon_type ?exon_start ?exon_end ?strand ?gene_name ?gene_description ?feat_name ?feat_description ?feat_class ?exon_name
FROM <http://togovar.org/ensembl>
WHERE {
  VALUES ?region_start { {{q.[1]}} }
  VALUES ?region_end { {{q.[2]}} }
  VALUES ?chr_id { <http://identifiers.org/hco/{{q.[0]}}#{{q.[3]}}> }

  # gene
  BIND("gene" AS ?gene_type)
  ?gene_id so:part_of ?chr_id .
  ?gene_id faldo:location ?gene_loc .
  ?gene_loc faldo:begin/a ?faldo_type .
  ?gene_loc faldo:begin/faldo:position ?gene_start_pos .
  ?gene_loc faldo:end/faldo:position ?gene_end_pos .

  FILTER(?faldo_type IN (faldo:ForwardStrandPosition, faldo:ReverseStrandPosition, faldo:BothStrandsPosition))
  BIND(IF (?faldo_type = faldo:ForwardStrandPosition, 1, IF (?faldo_type = faldo:ReverseStrandPosition, -1, 0)) AS ?strand)
  BIND(IF (?faldo_type = faldo:ReverseStrandPosition, ?gene_end_pos, ?gene_start_pos) AS ?gene_start)
  BIND(IF (!(?faldo_type = faldo:ReverseStrandPosition), ?gene_end_pos, ?gene_start_pos) AS ?gene_end)

  FILTER(!(?gene_start > ?region_end || ?gene_end < ?region_start))

  # transcript
  BIND("mRNA" AS ?feat_type)
  ?feat_id so:part_of ?gene_id .
  ?feat_id a ?feat_class_orig .
  FILTER(!strstarts(str(?feat_class_orig), "http://rdf.ebi.ac.uk/terms/ensembl/"))
  BIND(IF (?feat_class_orig = obo:SO_0001217, obo:SO_0000316, ?feat_class_orig) AS ?feat_class)

  ?feat_id faldo:location ?feat_loc .
  ?feat_loc faldo:begin/faldo:position ?feat_start_pos .
  ?feat_loc faldo:end/faldo:position ?feat_end_pos .
  BIND(IF (?faldo_type = faldo:ReverseStrandPosition, ?feat_end_pos, ?feat_start_pos) AS ?feat_start)
  BIND(IF (!(?faldo_type = faldo:ReverseStrandPosition), ?feat_end_pos, ?feat_start_pos) AS ?feat_end)

  # exon
  BIND("CDS" AS ?exon_type)
  ?feat_id so:has_part ?exon_id .

  ?exon_id faldo:location ?exon_loc .
  ?exon_loc faldo:begin/faldo:position ?exon_start_pos .
  ?exon_loc faldo:end/faldo:position ?exon_end_pos .
  BIND(IF (?faldo_type = faldo:ReverseStrandPosition, ?exon_end_pos, ?exon_start_pos) AS ?exon_start)
  BIND(IF (!(?faldo_type = faldo:ReverseStrandPosition), ?exon_end_pos, ?exon_start_pos) AS ?exon_end)

  OPTIONAL { ?gene_id dct:identifier ?gene_name . }
  OPTIONAL { ?gene_id rdfs:label ?gene_description . }
  OPTIONAL { ?feat_id dct:identifier ?feat_name . }
  OPTIONAL { ?feat_id so:translates_to/dct:identifier ?feat_description . }
  OPTIONAL { ?exon_id dct:identifier ?exon_name . }
}
```
