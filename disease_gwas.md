# Disease report / GWAS

## Parameters

* `medgen_cid` MedGen CID
  * default: C0023467
  * example: C0023467(acute myeloid leukemia), C2675520(breast-ovarian cancer)

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `medgen2efo`

```sparql
PREFIX dct:      <http://purl.org/dc/terms/>
PREFIX medgen:   <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX mo:       <http://med2rdf/ontology/medgen#>
PREFIX oboinowl: <http://www.geneontology.org/formats/oboInOwl#>
PREFIX rdfs:     <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?efo
WHERE {
  VALUES ?medgen { medgen:{{medgen_cid}} }

  GRAPH <http://togovar.biosciencedbc.jp/medgen> {
    ?medgen a mo:ConceptID ;
      mo:mgconso ?mgconso .

    ?mgconso dct:source mo:MONDO ;
      rdfs:seeAlso ?mondo .
  }

  GRAPH <http://togovar.biosciencedbc.jp/mondo> {
    ?mondo oboinowl:hasDbXref ?dbxref .
    FILTER STRSTARTS(STR(?dbxref), "EFO:")
    BIND(URI(CONCAT("http://www.ebi.ac.uk/efo/EFO_", SUBSTR(?dbxref, 5))) AS ?efo)
  }
}
```

## `efo`

```javascript
({medgen2efo}) => {
  return medgen2efo.results.bindings.map(b => b.efo.value);
}
```

## `efo2gwas`

```sparql
DEFINE sql:select-option "order"

PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX gwas:  <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?assoc ?variant_and_risk_allele ?raf ?p_value ?odds_ratio ?ci_text ?beta ?beta_unit ?pubmed_id
                ?pubmed AS ?pubmed_uri ?description AS ?study_detail ?study ?initial_sample_size ?replication_sample_size
WHERE {
  VALUES ?conditions { {{#each efo}} <{{this}}> {{/each}} }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog> {
    ?conditions ^terms:mapped_trait_uri ?assoc .

    ?assoc a gwas:Association ;
      terms:strongest_snp_risk_allele ?variant_and_risk_allele ;
      terms:risk_allele_frequency ?raf ;
      terms:p_value ?p_value ;
      terms:odds_ratio ?odds_ratio ;
      terms:beta ?beta ;
      # terms:beta_unit ?beta_unit ; # FIXME terms:beta_unit not found
      terms:ci_text ?ci_text ;
      terms:study ?study ;
      dct:references ?pubmed ;
      gwas:has_pubmed_id ?pubmed_id .

    ?study dct:description ?description ;
      terms:initial_sample_size ?initial_sample_size ;
      terms:replication_sample_size ?replication_sample_size .
  }
}
```

## `efo2traits`

```sparql
PREFIX gwas:  <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX terms: <http://med2rdf.org/gwascatalog/terms/>

SELECT DISTINCT ?assoc ?mapped_trait ?mapped_trait_uri
WHERE {
  VALUES ?mapped_trait_uri { {{#each efo}} <{{this}}> {{/each}} }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog> {
    ?assoc a gwas:Association ;
      terms:mapped_trait_uri ?mapped_trait_uri .
  }

  GRAPH <http://togovar.biosciencedbc.jp/efo> {
    ?mapped_trait_uri rdfs:label ?mapped_trait .
  }
}
```

## `rs2tgv`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, efo2gwas}) => {
  const num_of_rs_ids_per_fetch = 200;
  const rs = [...new Set(efo2gwas.results.bindings.flatMap(x => x.variant_and_risk_allele.value.split("; ").map(rs => rs.split('-')[0])))];
  const total = rs.length;

  return await [...Array(Math.ceil(total / num_of_rs_ids_per_fetch)).keys()].reduce(async (previousValue, currentValue, currentIndex) => {
    const prev = await previousValue;
    const start = currentIndex * num_of_rs_ids_per_fetch;
    const end = (currentIndex + 1) * num_of_rs_ids_per_fetch;
    const ret = await fetch(SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/rs2tgvid?rs_id=${encodeURIComponent(rs.slice(start, end).join(','))}`), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    }).then(res => res.json());

    return {...prev, ...ret};
  }, {});
}
```

## `result`

```javascript
({efo2gwas, efo2traits, rs2tgv}) => {
  const traits = {};

  efo2traits.results.bindings.forEach(x => {
    if (!traits[x.assoc.value]) {
      traits[x.assoc.value] = []
    }
    traits[x.assoc.value].push('<a href="' + x.mapped_trait_uri.value + '">' + x.mapped_trait.value + '</a>');
  })

  return efo2gwas.results.bindings.map(d => {
    const variant_and_risk_allele = d.variant_and_risk_allele.value.split(/;\s+/).map(rs_allele => {
      const rs_id = rs_allele.split('-')[0];
      const rs_risk_allele = rs_allele.split('-')[1];
      const tgv_id = rs2tgv[rs_id];
      const link_to_gwas = "<a href='https://www.ebi.ac.uk/gwas/variants/" + rs_id + "'>" + rs_id + "-" + rs_risk_allele + "</a>";
      const link_to_tgv = "<a href='/variant/" + rs2tgv[rs_id] + "'>" + rs2tgv[rs_id] + "</a>";

      return tgv_id ? "<span style='display: inline-block;'>" + link_to_gwas + "</span>(" + link_to_tgv + ")" : link_to_gwas;
    }).join('<br>');

    return {
      variant_and_risk_allele: variant_and_risk_allele,
      raf: d.raf.value != "NR" ? parseFloat(d.raf.value) : null,
      p_value: d.p_value.value != "NAN" ? parseFloat(d.p_value.value) : null,
      odds_ratio: d.odds_ratio.value != "NA" ? parseFloat(d.odds_ratio.value) : null,
      ci_text: d.ci_text.value,
      beta: d.beta.value != "NA" ? parseFloat(d.beta.value) : null,
      beta_unit: d.beta_unit?.value,
      mapped_trait: traits[d.assoc.value]?.join(),
      pubmed_id: d.pubmed_id.value,
      pubmed_uri: d.pubmed_uri.value,
      study_detail: d.study.value.replace("http://www.ebi.ac.uk/gwas/studies/", ""),
      study: d.study.value,
      initial_sample_size: d.initial_sample_size.value,
      replication_sample_size: d.replication_sample_size.value
    };
  });
}
```
