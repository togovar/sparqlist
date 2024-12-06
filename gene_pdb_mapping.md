# Structure data for TogoVar gene page

## Description

* Input
  * TogoVar の gene ページ用を目的としたため、HGNC ID スタート
* Output
  * structure: PDB dagta
    * pdb: PDB ID
    * chains: UniProt と対応する chain 
    * aligns: UniProt と PDB chain の配列一致範囲(begin, end)とポジションのズレ(delta)
      * positions 計算用、Stanza 側では使わない
    * positions: {"UniProt position": PDB chain position}
    * date: release date
    * resolution: high resolution
    * rfree: R-free
    * rwork: Rwork
  * varinat: TogoVar data
    * tgvid: TogoVar ID
    * label: Missense variant label
    * position: UniProt position
    * clinical_significance: Clinical significance from ClinVar

## Parameters

* `hgnc_id`
  * default: 404


## Endpoint

https://rdfportal.org/sib/sparql

## `uniprot`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX core: <http://purl.uniprot.org/core/>
PREFIX tax: <http://purl.uniprot.org/taxonomy/>
PREFIX hgnc: <http://purl.uniprot.org/hgnc/>
SELECT DISTINCT ?uniprot ?ensp ?enst ?isoform ?seq
WHERE {
  ?uniprot a core:Protein ;
           core:organism tax:9606 ;
           core:reviewed 1 ;
           rdfs:seeAlso hgnc:{{hgnc_id}} ;
           rdfs:seeAlso ?enst .
  ?enst core:translatedTo ?ensp .
  OPTIONAL {
    ?enst rdfs:seeAlso ?isoform .
    ?isoform a core:Simple_Sequence .
  }
}
```

## `ensp`
* ENSP の優先順位：1. core:Simple_sequence にリンクされている, 2. MANE select, 3. 先頭
```javascript
({uniprot}) => {
  // core:Simple_Sequence
  for (const d of uniprot.results.bindings) {
    if (d.isoform && d.isoform.value) {
      return {id: d.ensp.value.replace(/.+ensembl\.protein\//, "").replace(/\.\d+$/, "")};
    }
  }
  // mane
  let mane;
  for (const d of uniprot.results.bindings) {
    if (d.enst.value.match(/mane-select/)) {
      mane = d.enst.value.replace(/.+mane-select\//, "").replace(/\.\d+$/, "");
      break;
    }
  }
  for (const d of uniprot.results.bindings) {
    if (!d.enst.value.match(/mane-select/) && d.enst.value.match(mane)) {
      return {id: d.ensp.value.replace(/.+ensembl\.protein\//, "").replace(/\.\d+$/, "")};
    }
  }
  for (const d of uniprot.results.bindings) {
    if (!d.enst.value.match(/mane-select/)) {
      return {id: d.ensp.value.replace(/.+ensembl\.protein\//, "").replace(/\.\d+$/, "")};
    }
  }
  return {id: false};
}
```

## Endpoint

https://rdfportal.org/pdb/sparql

## `pdb_align`
* PDB のもろもろ情報
* UniProt と PDB cahin の alignment 位置関係情報
  * 複数に分割される場合あり
```sparql
PREFIX pdbo: <http://rdf.wwpdb.org/schema/pdbx-v50.owl#>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX up: <http://purl.uniprot.org/uniprot/>
SELECT DISTINCT ?pdb ?db_align_begin ?auth_align_begin ?auth_align_end ?resolution_high ?rfree ?rwork ?date (GROUP_CONCAT(DISTINCT ?chain_id ;  separator=',') AS ?chains)
WHERE {
{{#if ensp.id}}
  ?entry dct:identifier ?pdb ;
         pdbo:has_pdbx_audit_revision_historyCategory/pdbo:has_pdbx_audit_revision_history [
           pdbo:pdbx_audit_revision_history.ordinal "1" ;
           pdbo:pdbx_audit_revision_history.revision_date ?date 
         ] ;
         pdbo:has_entityCategory/pdbo:has_entity [
           pdbo:referenced_by_struct_ref [
             pdbo:link_to_uniprot <{{uniprot.results.bindings.0.uniprot.value}}> ; # UniProt
             pdbo:referenced_by_struct_ref_seq [
               pdbo:struct_ref_seq.db_align_beg ?db_align_begin ;              # 配列 align の UniProt の begin
               pdbo:struct_ref_seq.pdbx_auth_seq_align_beg ?auth_align_begin ; # 配列 align の PDB chain の begin
               pdbo:struct_ref_seq.pdbx_auth_seq_align_end ?auth_align_end     # 配列 align の PDB chain の end
             ]
           ] ;
           pdbo:referenced_by_struct_asym/pdbo:struct_asym.id ?chain_id
         ] .
  OPTIONAL {
    ?entry pdbo:has_refineCategory/pdbo:has_refine ?refine .
    OPTIONAL {
      ?refine pdbo:refine.ls_d_res_high ?resolution_high .
    }
    OPTIONAL {
      ?refine pdbo:refine.ls_R_factor_R_free ?rfree .
    }
    OPTIONAL {
      ?refine pdbo:refine.ls_R_factor_R_work ?rwork .
    }
  }
  # optional memo
  #   pdbo:refine.ls_d_res_high ?resolution_high ;
  #   pdbo:refine.ls_d_res_low ?resolution_low ;
  #   pdbo:refine.ls_R_factor_R_free ?rfree ;
  #   pdbo:refine.ls_R_factor_R_work ?rwork ;
  #   pdbo:refine.ls_R_factor_obs ?rfactor
{{/if}}
}
ORDER BY DESC(?date) ?pdb ?auth_align_begin
```

## `pdb_str`
* PDB 3D chain データにある position (?auth_pos)
```sparql
PREFIX pdbo: <http://rdf.wwpdb.org/schema/pdbx-v50.owl#>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX up: <http://purl.uniprot.org/uniprot/>
SELECT DISTINCT ?pdb ?auth_pos
WHERE {
{{#if ensp.id}}
  [] dct:identifier ?pdb ;
     pdbo:has_entityCategory/pdbo:has_entity [
       pdbo:referenced_by_struct_ref/pdbo:link_to_uniprot <{{uniprot.results.bindings.0.uniprot.value}}> ;
       pdbo:referenced_by_entity_poly/pdbo:referenced_by_entity_poly_seq/pdbo:referenced_by_pdbx_poly_seq_scheme/pdbo:pdbx_poly_seq_scheme.auth_seq_num ?auth_pos
     ] .
{{/if}}
}
ORDER BY ?pdb ?auth_pos
```

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `variants`
* TogoVar からの遺伝子上の variant 情報
```sparql
PREFIX rdfs:  <http://www.w3.org/2000/01/rdf-schema#>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
PREFIX hgnc:  <http://identifiers.org/hgnc/>
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>
SELECT DISTINCT ?tgvid ?hgvs ?clin_sig
WHERE {
{{#if ensp.id}}
  VALUES ?hgnc { hgnc:{{hgnc_id}} }

  GRAPH <http://togovar.org/hgnc> {
    ?hgnc rdfs:label ?symbol .
  }
    
  ?var tgvo:hasConsequence [
    tgvo:hgvsp ?hgvs ;
    tgvo:gene/rdfs:label ?symbol
  ] .
  FILTER (!REGEX (?hgvs, "=")) # filtering synonimous

  GRAPH <http://togovar.org/variant>{
    ?var dct:identifier ?tgvid.
  }
  OPTIONAL {
    GRAPH <http://togovar.org/variant/annotation/clinvar> {
      ?var dct:identifier ?var_id .
    }
    BIND(IRI(CONCAT("http://ncbi.nlm.nih.gov/clinvar/variation/", ?var_id)) AS ?clinvar)
    #?clinvar cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession/cvo:interpretation ?clin_sig.
    ?clinvar cvo:classified_record/cvo:rcv_list/cvo:rcv_accession/cvo:rcv_classifications/cvo:germline_classification/cvo:description/cvo:description ?clin_sig.
  }
{{/if}}
}
```

## `return`
```javascript
async ({hgnc_id, uniprot, ensp, pdb_align, pdb_str, variants})=>{
  if (! ensp.id) return {structure: [], variant: []};
  const ensp_id = ensp.id;

  let res = {structure: [], variant: []};

  // PDB
  let pdb_d = {};
  for (const pdb of pdb_align.results.bindings) {
    const id = pdb.pdb.value;
    if (! pdb_d[id]) {
      res.structure.push( 
        {
          pdb: id,
          date: pdb.date.value,
          chains: pdb.chains.value.split(","),
          resolution: pdb.resolution_high ? pdb.resolution_high.value : null,
          rfree: pdb.rfree ? pdb.rfree.value : null,
          rwork: pdb.rwork ? pdb.rwork.value : null,
          aligns: [],
          positions: {}
        }
      );
      pdb_d[id] = res.structure.length; // pdb の要素番号 + 1
    }
    // UniProt と PDB chain の alignemt 取れている範囲とそのずれ(delta)
    const d = pdb_d[id] - 1;
    res.structure[d].aligns.push(
      {
        begin: parseInt(pdb.auth_align_begin.value),
        end: parseInt(pdb.auth_align_end.value),
        delta: parseInt(pdb.db_align_begin.value) - parseInt(pdb.auth_align_begin.value)
      }
    );     
  }

  // PDB chain 3D position で、上記 alignment に入っているものだけに絞り込み
  for (const pdb of pdb_str.results.bindings) {
    const id = pdb.pdb.value;
    if (! pdb_d[id]) continue;
    const d = pdb_d[id] - 1;
    for (const align of res.structure[d].aligns) {
      if (align.begin <= parseInt(pdb.auth_pos.value) && parseInt(pdb.auth_pos.value) <= align.end) {
        const db_pos = parseInt(pdb.auth_pos.value) + align.delta;
        res.structure[d].positions[db_pos.toString()] = parseInt(pdb.auth_pos.value);
      }
    }
  }
  // AlphaFold
  res.structure.push(
    {
      pdb: "AlphaFold",
      url: "https://alphafold.ebi.ac.uk/files/AF-" + uniprot.results.bindings[0].uniprot.value.replace(/.+uniprot\//, "") + "-F1-model_v4.pdb",
      uniprot: uniprot.results.bindings[0].uniprot.value.replace(/.+uniprot\//, ""),
      chains: ["A"]
    }
  );
  
  // Variant
  for (const variant of variants.results.bindings) {
    if (variant.hgvs.value.match(ensp_id)) {
      const label = variant.hgvs.value.replace(/.+:p\./, "");
      const position = label.match(/[^\d]+(\d+)/)[1];
      res.variant.push(
        {
          tgvid: variant.tgvid.value,
          label: label,
          position: parseInt(position),
          aa: label.match(/^[A-Z][a-z][a-z]/) ? label.match(/^([A-Z][a-z][a-z])/)[1].toUpperCase() : null,
          clinical_significance: variant.clin_sig ? variant.clin_sig.value : null
        }
      )
    }
  }
  res.variant = res.variant.sort((a, b) => {
    return a.position > b.position ? 1 : -1;  
  });
  return res;
}
```
