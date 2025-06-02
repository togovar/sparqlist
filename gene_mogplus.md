# Convert human variants to mouse genome positions per gene

## Parameters

* `hgnc_id` : hgnc_id
  * example: 404 1101 1102 11998
* `mogplus_ver` : version
  * default: mogplus21
  * example: mogplus3

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `symbol`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX hgnc: <http://identifiers.org/hgnc/>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>
SELECT DISTINCT ?symbol
WHERE {
  VALUES ?hgnc_uri { hgnc:{{hgnc_id}} }
  ?hgnc_uri dct:identifier ?hgnc_id ;
            rdfs:label ?symbol .
}
```

## Endpoint

https://rdfportal.org/ebi/sparql

## `range`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX ensterm: <http://rdf.ebi.ac.uk/terms/ensembl/>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX so: <http://purl.obolibrary.org/obo/so#>
PREFIX tax: <http://identifiers.org/taxonomy/>
SELECT DISTINCT ?chr (MIN(?b) AS ?begin) (MAX(?e) AS ?end)
FROM <http://rdfportal.org/dataset/ensembl>
WHERE {
  [] a ensterm:EnsemblGene ;
     rdfs:label "{{symbol.results.bindings.0.symbol.value}}" ;
     obo:RO_0002162 tax:9606 ;
     so:part_of ?chr ;
     faldo:location [
       faldo:begin / faldo:position ?b ;
       faldo:end / faldo:position ?e 
     ] .
  FILTER (REGEX (STR (?chr), "identifiers.org"))
}
```

## `clinvar`
```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, SPARQLIST_TOGOVAR_APP, hgnc_id})=>{
  const clinvar = await fetch("/sparqlist/api/gene_clinvar?hgnc_id=" + hgnc_id, {
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
    },
  }).then(res => res.json());
  return clinvar;
}
```

## `sequence`
```javascript
function sort_consequences(consequence_str) {
  const consequence_order = [
    "transcript_ablation",
    "splice_acceptor_variant",
    "splice_donor_variant",
    "stop_gained",
    "frameshift_variant",
    "stop_lost",
    "start_lost",
    "transcript_amplification",
    "feature_elongation",
    "feature_truncation",
    "inframe_insertion",
    "inframe_deletion",
    "missense_variant",
    "protein_altering_variant",
    "splice_donor_5th_base_variant",
    "splice_region_variant",
    "splice_donor_region_variant",
    "splice_polypyrimidine_tract_variant",
    "incomplete_terminal_codon_variant",
    "start_retained_variant",
    "stop_retained_variant",
    "synonymous_variant",
    "coding_sequence_variant",
    "mature_miRNA_variant",
    "5_prime_UTR_variant",
    "3_prime_UTR_variant",
    "non_coding_transcript_exon_variant",
    "intron_variant",
    "NMD_transcript_variant",
    "non_coding_transcript_variant",
    "coding_transcript_variant",
    "upstream_gene_variant",
    "downstream_gene_variant",
    "TFBS_ablation",
    "TFBS_amplification",
    "TF_binding_site_variant",
    "regulatory_region_ablation",
    "regulatory_region_amplification",
    "regulatory_region_variant",
    "intergenic_variant",
    "sequence_variant"
  ];

  return consequence_str
    .split(',')
    .map(s => s.trim())
    .sort((a, b) => {
      const index_a = consequence_order.indexOf(a);
      const index_b = consequence_order.indexOf(b);
      return index_a - index_b;
    })
    .join('<br/>');
}

function get_clinvar(tgv_id, rs_id, clinvars_by_gene) {
  let clinvars_by_rs_id = [];

  for (let entry of clinvars_by_gene) {
    if (entry.tgv_id == tgv_id ||
        entry.tgv_id == "" && entry.rs_id == rs_id) {
      let condition = "";
      if (entry.medgen) {
        condition = "<a href='/disease/"
          + entry.medgen.replace("https://www.ncbi.nlm.nih.gov/medgen/", "")
          + "'>" + entry.condition + "</a>";
      } else {
        condition = entry.condition;
      }

      clinvars_by_rs_id.push({
        interpretation_order: entry.interpretation_order,
        interpretation: entry.interpretation,
        condition: condition
      });
    }
  }

  clinvars_by_rs_id.sort((a, b) => {
    const order_a = a.interpretation_order;
    const order_b = b.interpretation_order;
    return order_a - order_b;
  });

  const most_sig_interpretation = Math.min(...clinvars_by_rs_id.map(e => e.interpretation_order));
  const interpretations = clinvars_by_rs_id.map(e => e.interpretation).join("<br/>");
  const conditions = clinvars_by_rs_id.map(e => e.condition).join("<br/>");

  return [interpretations, most_sig_interpretation, conditions];
}

async ({SPARQLIST_TOGOVAR_SPARQLIST, SPARQLIST_TOGOVAR_APP, mogplus_ver, symbol, range, clinvar})=>{
  if (! symbol.results.bindings[0]) return "Gene symbol not found or invalid.";
  if (! range.results.bindings[0]) return "Genomic position not found for gene symbol " + symbol.results.bindings[0];

  const chr = range.results.bindings[0].chr.value.replace("http://identifiers.org/hco/", "").replace("/GRCh38","");
  const begin  = range.results.bindings[0].begin.value;
  const end    = range.results.bindings[0].end.value;  

  // make strain list
  const strain2id = await fetch("/sparqlist/api/mouse_strain?strain_id=&strain=all", {
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
    },
  }).then(res => res.json());

  // convert strand
  const conv_nt = (strand, nt) => {
    if (strand == "+") return nt;
    if (nt == "A") return "T";
    if (nt == "T") return "A";
    if (nt == "C") return "G";
    if (nt == "G") return "C";
  }

  let api = "https://sparql-support.dbcls.jp/api/getTogoVarGeneAnn?q=";
  let position = chr + ":" + begin + "-" + end;
  const togovar = await fetch(api + position).then(d=>d.text());

  // GRCh38 => GRCm39(mm39)  
  api = "https://orth.dbcls.jp/cgi-bin/liftOver/";
  position = "/chr" + position;
  let mmu_chr, mmu_start, mmu_end, mmu_strand;
  const map = "hg38ToMm39";
  const liftover = await fetch(api + map + position).then(d=>d.text());
  const list = liftover.split(/\n/);
  let hsa2mmu = {};
  for (const l of list) {
    if (l.match(/^hg/)) {
      let [,,,hsa_pos,,mmu_chr_t, strand, mmu_pos] = l.split(/\s/);
      if (!mmu_chr) mmu_chr = mmu_chr_t.replace(/chr/, "");
      if (!mmu_strand) mmu_strand = strand;
      mmu_pos = parseInt(mmu_pos);
      if (!mmu_start) mmu_start = mmu_pos;
      mmu_end = mmu_pos;
      hsa2mmu[hsa_pos] = mmu_pos;
    }
  }
  if (!mmu_start) return "Liftover position not found";

  // MoG+ GRCm389(mm39) variants
  const options = {method: 'GET', headers: {'Accept': 'application/json'}};
  const mmu_strains = await fetch("/sparqlist/api/mouse_strain?strain_id=all", options).then(d => d.json());
  let strain_ids = [];
  for (const d of Object.keys(mmu_strains)) {
    if ((mmu_strains[d].category == "mogplus3" && mogplus_ver == "mogplus21")
        || (mmu_strains[d].category == "mogplus21" && mogplus_ver == "mogplus3")) continue;
    strain_ids.push(encodeURIComponent(mmu_strains[d].id));
  }
  if (mmu_end < mmu_start) {
    let tmp = mmu_end;
    mmu_end = mmu_start;
    mmu_start = tmp;
  }
  const mog_body = "strainNoSlct=" + strain_ids.join("&strainNoSlct=") + "&"
    + "chrName=" + mmu_chr + "&chrStart=" + mmu_start + "&chrEnd=" + mmu_end + "&"
    + "&seqType=genome&chrName=5&geneNameSearchText=&index=submit&presentType=dwnld";
  const mogp = await fetch("https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?" + mog_body).then(d=>d.text());
  const mogp_res = mogp.split(/\n/);
  let pos_list = mogp_res[0].split(/\t/);
  let var_list = mogp_res[1].split(/\t/);
  if (!pos_list[1]) return "MoG+ variant not found";

  let mmu_var_pos = {};
  let col2pos = [];
  for (let i = 1; i < pos_list.length; i++) {
    col2pos[i] = pos_list[i];
    // mog+ variant view region
    let start = i - 20;
    let end = i + 20;
    if (start < 1) start = 1;
    if (end > pos_list.length - 1) end = pos_list.length -1;
    start = pos_list[start];
    end = pos_list[end];
    if (end < start) {
      let tmp = end;
      end = start;
      start = tmp;
    }    
    mmu_var_pos[pos_list[i]] = {
      ref: var_list[i],
      alt: [],
      strain: {},
      region: "&chrStart=" + start + "&chrEnd=" + end
    };
  }
  for (let i = 2; i < mogp_res.length; i++) {
    var_list = mogp_res[i].split(/\t/);
    for (let j = 1; j < var_list.length; j++) {
      if (var_list[j].match(/[ATCG-]/)) {
        let pos = col2pos[j];
        if (!mmu_var_pos[pos].alt.includes(var_list[j])) mmu_var_pos[pos].alt.push(var_list[j]);
        (mmu_var_pos[pos].strain[var_list[j]] ??= []).push(var_list[0]);
      }
    }
  }

  // data 
  let r = [];
  togovar.split("\n").forEach(d => {
    const [tgv_id, rs, chr, hsa_pos, hsa_ref, hsa_alt, symbol, transcript_id, consequence, sift_qualitative_prediction, 
           sift_score, polyphen2_qualitative_prediction, polyphen2_score, alphamissense_pathogenicity, alphamissense_score] = d.split(/\t/);
    const mmu_pos = hsa2mmu[hsa_pos];
    let mmu_ref;
    if (mmu_var_pos[mmu_pos]?.ref) mmu_ref = conv_nt(mmu_strand, mmu_var_pos[mmu_pos].ref);
    if (mmu_ref == hsa_ref && mmu_var_pos[mmu_pos]?.alt[0] && hsa_ref.length == 1 && hsa_alt.length == 1) {
      for (const mmu_alt of mmu_var_pos[mmu_pos].alt) {
        let alt_match = false;
        if (hsa_alt == conv_nt(mmu_strand, mmu_alt)) alt_match = true;
        let mouse_strains = [];
	      let mouse_ids = [];
        for (let strain_name of mmu_var_pos[mmu_pos].strain[mmu_alt]){
	        mouse_ids.push(strain2id[strain_name].id);
          if(strain2id[strain_name]?.source){
            mouse_strains.push("<a href=" + strain2id[strain_name].source + ">" + strain_name + "</a>");
          }else{
            mouse_strains.push(strain_name);
          }
        }

        const [interpretation, most_sig_interpretation, condition] = get_clinvar(tgv_id, rs, clinvar)

        r.push({
          tgv_id: tgv_id == "-" ? "" : tgv_id,
          tgv_link: tgv_id == "-" ? "" : "/variant/" + tgv_id,
          rs: rs == "-" ? "" : rs,
          rs_link: rs == "-" ? "" : "https://www.ncbi.nlm.nih.gov/snp/" + rs,
          allele_grch38: chr + ":" + hsa_pos + "-" + hsa_ref + "-" + hsa_alt,
          consequence: sort_consequences(consequence),
          most_sig_interpretation: most_sig_interpretation,
          clinsig: interpretation ? interpretation : "",
          condition: condition ? condition : "",
          mmu_strand: mmu_strand,
          allele_grcm39: mmu_chr + ":" + mmu_pos + "-" + mmu_ref + "-" + mmu_alt,
          ref_grcm39: mmu_ref,
          alt_grcm39: mmu_alt,
          alt_match: alt_match == true ? "Yes" : "No",
          mouse_strains: mouse_strains.join("<br/>"),
          mogplus_url: "https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?strainNoSlct=refGenome&strainNoSlct="
          + mouse_ids.join("&strainNoSlct=").replace(/\//g, "_") + "&chrName=" + mmu_chr + mmu_var_pos[mmu_pos].region
          + "&seqType=genome&chrName=5&geneNameSearchText=&index=submit&presentType=disp"
        })
      }
    }
  });

  r.sort((a, b) => {
    if (a.clinsig === "" && b.clinsig !== "") return 1;
    if (a.clinsig !== "" && b.clinsig === "") return -1;
    return a.most_sig_interpretation - b.most_sig_interpretation;
  });

  return r;
}
```
