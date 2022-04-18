# Gene report / GWAS

## Parameters

* `hgnc_id` HGNC ID
  * default: 404

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `gene2gwas`

```sparql
DEFINE sql:select-option "order"

PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX gwas:  <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT DISTINCT ?assoc ?variant_and_risk_allele ?rs_id ?raf ?p_value ?odds_ratio ?ci_text ?beta ?beta_unit ?pubmed_id 
                ?pubmed AS ?pubmed_uri ?description AS ?study_detail ?study ?initial_sample_size ?replication_sample_size
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{hgnc_id}}> }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    ?hgnc_uri rdfs:label ?gene_symbol .
  }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog> {
    ?assoc terms:reported_genes ?gene_symbol ;
      a gwas:Association ;
      terms:strongest_snp_risk_allele ?variant_and_risk_allele ;
      terms:risk_allele_frequency ?raf ;
      terms:snps ?rs_id ;
      terms:p_value ?p_value ;
      terms:odds_ratio ?odds_ratio ;
      terms:beta ?beta ;
      terms:beta_unit ?beta_unit ;
      terms:ci_text ?ci_text ;
      terms:study ?study ;
      dct:date ?association_date ;
      dct:references ?pubmed ;
      gwas:has_pubmed_id ?pubmed_id .

    ?study dct:identifier ?study_id ;
      dct:description ?description ;
      terms:initial_sample_size ?initial_sample_size ;
      terms:replication_sample_size ?replication_sample_size .
  }
}
```

## `gene2traits`

```sparql
PREFIX gwas:  <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT DISTINCT ?assoc ?mapped_trait ?mapped_trait_uri
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{hgnc_id}}> }

  GRAPH <http://togovar.biosciencedbc.jp/hgnc> {
    ?hgnc_uri rdfs:label ?gene_symbol .
  }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog> {
    ?assoc terms:reported_genes ?gene_symbol ;
      a gwas:Association ;
      terms:mapped_trait_uri ?mapped_trait_uri .
  }

  GRAPH <http://togovar.biosciencedbc.jp/efo> {
    ?mapped_trait_uri rdfs:label ?mapped_trait .
  }
}
```

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_SEARCH, gene2gwas, gene2traits}) => {
  const traits = {};

  gene2traits.results.bindings.map(x => {
    if (!traits[x.assoc.value]) {
      traits[x.assoc.value] = []
    }
    traits[x.assoc.value].push('<a href="' + x.mapped_trait_uri.value + '">' + x.mapped_trait.value + '</a>');
  })

  const rs = [...new Set(gene2gwas.results.bindings.map(x => x.rs_id.value))];
  const variant_info = await fetch(SPARQLIST_TOGOVAR_SEARCH.concat("?stat=0&quality=0&term=", encodeURIComponent(rs.join(','))), {
    method: 'GET',
    headers: {
      'Accept': 'application/json',
    }
  }).then(res => {
    return res.json();
  }).then(json => {
    const result = {};

    json.data.forEach(x => {
      const rs = x.existing_variations[0];

      if (rs) {
        result[rs] = {
          tgv_id: x.id,
          position: x.chromosome + ":" + x.start,
          ref_alt: `<div class="ref-alt"><span class="ref" data-sum="${x.reference.length}">${x.reference}</span><span class="arrow"></span><span class="alt" data-sum="${x.alternative.length}">${x.alternative}</span></div>`,
          alt_freq_gem_j_wga: x.frequencies?.filter(f => f.source === 'gem_j_wga')?.[0]?.allele?.frequency,
          raf: x.raf,
        };
      }
    });

    return result;
  });

  return gene2gwas.results.bindings.map(x => ({
    tgv_id: variant_info[x.rs_id.value]?.tgv_id,
    tgv_link: variant_info[x.rs_id.value] ? `/variant/${variant_info[x.rs_id.value].tgv_id}` : "",
    variant_and_risk_allele: x.variant_and_risk_allele.value,
    rs_uri: "https://www.ebi.ac.uk/gwas/variants/" + x.rs_id.value,
    position: variant_info[x.rs_id.value]?.position,
    ref_alt: variant_info[x.rs_id.value]?.ref_alt,
    alt_freq: variant_info[x.rs_id.value]?.alt_freq_gem_j_wga,
    raf: x.raf.value != "NR" ? parseFloat(x.raf.value) : null,
    p_value: x.p_value.value != "NAN" ? parseFloat(x.p_value.value) : null,
    odds_ratio: x.odds_ratio.value != "NA" ? parseFloat(x.odds_ratio.value) : null,
    ci_text: x.ci_text.value,
    beta: x.beta.value != "NA" ? parseFloat(x.beta.value) : null,
    beta_unit: x.beta_unit?.value,
    mapped_trait: traits[x.assoc.value]?.join(),
    pubmed_id: x.pubmed_id.value,
    pubmed_uri: x.pubmed_uri.value,
    study_detail: x.study.value.replace("http://www.ebi.ac.uk/gwas/studies/", ""),
    study: x.study.value,
    initial_sample_size: x.initial_sample_size.value,
    replication_sample_size: x.replication_sample_size.value
  }));
}
```
