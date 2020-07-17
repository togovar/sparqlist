# jBrowse Gene Track

## Parameters

* `query` {chromosome}:{start}:{end}@{endpoint}
  * default: 12:112200000:112250000@https://togovar.biosciencedbc.jp/sparql

## `ep` Extract endpoint URL

```javascript
({
  json({query}) {
    return query.split('@')[1] || 'https://togovar.biosciencedbc.jp/sparql';
  }
})
```

## Endpoint

{{ep}}

## `chromosome` Extract chromosome

```javascript
({
  json({query}) {
    return query.split(':')[0];
  }
})
```

## `start` Extract start position

```javascript
({
  json({query}) {
    return query.split(':')[1];
  }
})
```

## `end` Extract end position

```javascript
({
  json({query}) {
    return query.split(':')[2].replace(/@.*/, '');
  }
})
```

## `result`

```sparql
DEFINE sql:select-option "order"

PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX so: <http://purl.obolibrary.org/obo/so#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX sio: <http://semanticscience.org/resource/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>

SELECT DISTINCT ?gene_id ?gene_type ?gene_start ?gene_end ?feat_id ?feat_type ?feat_start ?feat_end ?exon_id ?exon_type ?exon_start ?exon_end ?strand ?gene_name ?gene_description ?feat_name ?feat_description ?feat_class ?exon_name
FROM <http://togovar.biosciencedbc.jp/ensembl37>
WHERE {
  VALUES ?region_start { {{start}} }
  VALUES ?region_end { {{end}} }
  VALUES ?chr_id { <http://identifiers.org/hco/{{chromosome}}#GRCh37> }

  # gene
  BIND ("gene" AS ?gene_type)
  ?gene_id so:part_of ?chr_id .
  ?gene_id faldo:location ?gene_loc .
  ?gene_loc faldo:begin/rdf:type ?faldo_type .
  ?gene_loc faldo:begin/faldo:position ?gene_start_pos .
  ?gene_loc faldo:end/faldo:position ?gene_end_pos .

  FILTER (?faldo_type IN (faldo:ForwardStrandPosition, faldo:ReverseStrandPosition, faldo:BothStrandsPosition))
  BIND (IF (?faldo_type = faldo:ForwardStrandPosition, 1, IF (?faldo_type = faldo:ReverseStrandPosition, -1, 0)) AS ?strand)
  BIND (IF (?faldo_type = faldo:ReverseStrandPosition, ?gene_end_pos, ?gene_start_pos) AS ?gene_start)
  BIND (IF (!(?faldo_type = faldo:ReverseStrandPosition), ?gene_end_pos, ?gene_start_pos) AS ?gene_end)

  FILTER (!(?gene_start > ?region_end || ?gene_end < ?region_start))

  # transcript
  BIND ("mRNA" AS ?feat_type)
  ?feat_id so:part_of ?gene_id .
  ?feat_id rdf:type ?feat_class_orig .
  FILTER (!strstarts(str(?feat_class_orig), "http://rdf.ebi.ac.uk/terms/ensembl/"))
  BIND (IF (?feat_class_orig = obo:SO_0001217, obo:SO_0000316, ?feat_class_orig) AS ?feat_class)

  ?feat_id faldo:location ?feat_loc .
  ?feat_loc faldo:begin/faldo:position ?feat_start_pos .
  ?feat_loc faldo:end/faldo:position ?feat_end_pos .
  BIND (IF (?faldo_type = faldo:ReverseStrandPosition, ?feat_end_pos, ?feat_start_pos) AS ?feat_start)
  BIND (IF (!(?faldo_type = faldo:ReverseStrandPosition), ?feat_end_pos, ?feat_start_pos) AS ?feat_end)

  # exon
  BIND ("CDS" AS ?exon_type)
  ?feat_id so:has_part ?exon_id .

  ?exon_id faldo:location ?exon_loc .
  ?exon_loc faldo:begin/faldo:position ?exon_start_pos .
  ?exon_loc faldo:end/faldo:position ?exon_end_pos .
  BIND (IF (?faldo_type = faldo:ReverseStrandPosition, ?exon_end_pos, ?exon_start_pos) AS ?exon_start)
  BIND (IF (!(?faldo_type = faldo:ReverseStrandPosition), ?exon_end_pos, ?exon_start_pos) AS ?exon_end)

  OPTIONAL { ?gene_id dct:identifier ?gene_name . }
  OPTIONAL { ?gene_id rdfs:label ?gene_description . }
  OPTIONAL { ?feat_id dct:identifier ?feat_name . }
  OPTIONAL { ?feat_id so:translates_to/dct:identifier ?feat_description . }
  OPTIONAL { ?exon_id dct:identifier ?exon_name . }
}
```
