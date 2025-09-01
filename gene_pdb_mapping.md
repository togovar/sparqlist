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
    * id: TogoVar ID or dbSNP rs ID
    * label: Missense variant label (HGVS_p)
    * position: UniProt position
    * sig: significance code
    * sig_label: Clinical significance label
    * color: cignificance color
    * color_num: color score
    

## Parameters

* `hgnc_id`
  * default: 404
* `togovar_api`
  * default: https://stg-grch38.togovar.org/api/search/variant

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

## `return`
```javascript
async ({hgnc_id, togovar_api, uniprot, ensp, pdb_align, pdb_str})=>{
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
  const sig2label = {
    P: {label: "Pathogenic", score: 0, color: 0},
    LP: {label: "Likely pathogenic", score: 1, color: 0},
    PLP: {label: "Pathogenic Low penetrance", score: 2, color: 1},
    LPLP: {label: "Likely pathogenic, Low penetrance", score: 3, color: 1},
    ERA: {label: "Established risk allele", score: 4, color: 2},
    LRA: {label: "Likely risk allele", score: 5, color: 2},
    URA: {label: "Uncertain risk allele", score: 6, color: 2},
    US: {label: "Uncertain significance", score: 7, color: 3},
    LB: {label: "Likely benign", score: 8, color: 4},
    B: {label: "Benign", score: 9, color: 5},
    CI: {label: "Conflicting interpretations of pathogenicity", score: 10, color: 6},
    DR: {label: "Drug response", score: 11, color: 7},
    CS: {label: "Confers sensitivity", score: 12, color: 7},
    RF: {label: "Risk factor", score: 13, colre: 7},
    A: {label: "Association", score: 14, color: 7},
    PR: {label: "Protective", score: 15, color: 7},
    AF: {label: "Affects", score: 16, color: 7},
    O: {label: "Other", score: 17, color: 8},
    NP: {label: "Not provided", score: 18, color: 8},
    AN: {label: "Association not found", score: 19, color: 8}
  };
  const num2color = ["255,90,84", "255,149,60", "255,174,0", "187,186,126", "157,207,58", "4,175,88", "198,84,255", "156,130,0", "148,146,141"];
  const tgv_bdy = `{"offset":#offset,"limit":#limit,"query":{"and":[{"gene":{"relation":"eq","terms":[#hgncid]}},
    {"consequence":{"relation":"eq","terms":["missense_variant","frameshift_variant"]}}]}}`;
  let tgv_opt = {method: 'POST', headers: {'Accept': 'application/json', 'Content-Type': 'application/json'}}

  let filtered = false;
  const limit = 500;
  let offset = 0;
  let count = 0;
  while (!filtered || filtered > count) {
    tgv_opt.body = tgv_bdy.replace(/#hgncid/, hgnc_id).replace(/#offset/, offset).replace(/#limit/, limit);
    const togovar = await fetch(togovar_api, tgv_opt).then(res => res.json());
    filtered = togovar.statistics?.filtered;
    if (filtered == 0) break;
    if (togovar.data) {
      for (const v of togovar.data) {
        let id = "";
        if (v.id) id = v.id;
        else if (v.existing_variations) id = v.existing_variations[0];
        if (!id || !v.transcripts) continue;
        let pos, hgvs_p;
        for (const t of v.transcripts) {
          if (t.hgvs_p?.match(ensp_id)) {
            if (t.hgvs_p.match(/:p\.[A-Z][a-z]{2}\d+.+/) && !t.hgvs_p.match(/=/)) {
              [, hgvs_p, pos] = t.hgvs_p.match(/:p\.([A-Z][a-z]{2}(\d+).+)/);
              break;
            }
          }
        }
        if (!pos || !hgvs_p) break;

	let sig = false;
        if (v.significance) {
          let min = 99;
          for (const s of v.significance) {
            if (!s.interpretations[0]) continue;
            if (sig2label[s.interpretations[0]].score < min) {
              min = sig2label[s.interpretations[0]].score;
              sig = s.interpretations[0];
            }
          }
	}
        res.variant.push(
          {
            id: id,
            position: parseInt(pos),
            label: hgvs_p,
            ...(sig && {sig: sig}),
            ...(sig && {sig_label: sig2label[sig].label}),
            ...(sig && {color: num2color[sig2label[sig].color]}),
            ...(sig && {color_num: sig2label[sig].color})
          }
        );
      }
    }
    offset = '["' + togovar.data[togovar.data.length - 1].chromosome.replace("X", "23").replace("Y", "24").replace("MT", "25") + '","' + togovar.data[togovar.data.length - 1].position + '","' + togovar.data[togovar.data.length - 1].reference + '","' + togovar.data[togovar.data.length - 1].alternate + '"]';
    count += limit;
  }
  res.variant = res.variant.sort((a, b) => {
    return a.position > b.position ? 1 : -1;  
  });
  return res;
}
```
