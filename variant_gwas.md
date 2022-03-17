# Variant report / GWAS

## Parameters

* `tgv_id` TogoVar ID
  * default: tgv83272255
  * example: tgv6252547

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `xref`

```sparql
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?xref
WHERE {
  VALUES ?tgv_id { "{{tgv_id}}" }

  GRAPH <http://togovar.biosciencedbc.jp/variant> {
    ?variant dct:identifier ?tgv_id .
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant/annotation/ensembl> {
    ?variant rdfs:seeAlso ?xref .
  }
}
```

## `rs`

```javascript
({xref}) => {
  const prefix = "http://identifiers.org/dbsnp/";

  return xref.results.bindings.filter(x => x.xref.value.startsWith(prefix)).map(x => x.xref.value.replace(prefix, ""));
}
```

## `rs2gwas`

```sparql
DEFINE sql:select-option "order"

PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX gwas:  <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT ?assoc ?variant_and_risk_allele ?rs ?raf ?p_value ?odds_ratio ?ci_text ?beta ?beta_unit
       ?pubmed_id ?pubmed_uri ?study_detail ?study ?initial_sample_size ?replication_sample_size
WHERE {
  VALUES ?rs { {{#each rs}} "{{this}}" {{/each}} }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog> {
    ?assoc terms:snps ?rs ;
      a gwas:Association ;
      terms:reported_genes ?reported_genes ;
      terms:strongest_snp_risk_allele ?variant_and_risk_allele ;
      terms:risk_allele_frequency ?raf ;
      terms:p_value ?p_value ;
      terms:odds_ratio ?odds_ratio ;
      terms:beta ?beta ;
      # terms:beta_unit ?beta_unit ; # FIXME terms:beta_unit not found
      terms:ci_text ?ci_text ;
      terms:study ?study ;
      dct:date ?association_date ;
      dct:references ?pubmed_uri ;
      gwas:has_pubmed_id ?pubmed_id .

    ?study dct:identifier ?study_id ;
      dct:description ?study_detail ;
      terms:initial_sample_size ?initial_sample_size ;
      terms:replication_sample_size ?replication_sample_size .
  }
}
ORDER by ?p_value
```

## `rs2traits`

```sparql
PREFIX gwas:  <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT ?assoc ?mapped_trait ?mapped_trait_uri
WHERE{
  VALUES ?rs { {{#each rs}} "{{this}}" {{/each}} }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog> {
    ?assoc a gwas:Association ;
      terms:snps ?rs ;
      terms:mapped_trait_uri ?mapped_trait_uri .
  }

  GRAPH <http://togovar.biosciencedbc.jp/efo> {
    ?mapped_trait_uri rdfs:label ?mapped_trait .
  }
}
```

## `trait_html`

```javascript
({rs2traits}) => {
  const traits = {};

  rs2traits.results.bindings.map(x => {
    if (!traits[x.assoc.value]) {
      traits[x.assoc.value] = []
    }
    traits[x.assoc.value].push('<a href="' + x.mapped_trait_uri.value + '">' + x.mapped_trait.value + '</a>');
  })

  return traits;
}
```

## `result`

```javascript
({rs2gwas, trait_html}) => {
  return rs2gwas.results.bindings.map(d => {
    return {
      variant_and_risk_allele: d.variant_and_risk_allele.value,
      rs_uri: "https://www.ebi.ac.uk/gwas/variants/" + d.rs.value,
      raf: d.raf.value,
      p_value: d.p_value.value,
      odds_ratio: d.odds_ratio.value,
      ci_text: d.ci_text.value,
      beta: d.beta.value,
      beta_unit: d.beta_unit?.value,
      mapped_trait: trait_html[d.assoc.value]?.join(),
      pubmed_id: d.pubmed_id.value,
      pubmed_uri: d.pubmed_uri.value,
      study_detail: d.study.value.replace("http://www.ebi.ac.uk/gwas/studies/", ""),
      study: d.study.value,
      initial_sample_size: d.initial_sample_size.value,
      replication_sample_size: d.replication_sample_size.value,
    };
  });
}
```
