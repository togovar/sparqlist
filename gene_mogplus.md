# Convert human variants to mouse genome positions per gene

## Parameters

* `hgnc_id` : hgnc_id
  * example: 404
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
  const strain2id = await fetch("/sparqlist/api/gene_clinvar?hgnc_id=" + hgnc_id, {
    headers: {
      Accept: 'application/json',
      'Content-Type': 'application/json',
    },
  }).then(res => res.json());
  return strain2id;
}
```

## `sequence`
```javascript
function get_clinsig(rs, clinvar){
  for(const entry of clinvar){
    let interpretation;
    let condition;
    let condition_link;
    if(entry.rs_id == rs){
      if(entry.medgen){
        condition_link = entry.medgen.replace("https://www.ncbi.nlm.nih.gov/medgen/","/disease/")
      }
      ret = [entry.interpretation, entry.condition, condition_link];
      return ret;
    }
  }

  return [null, null, null];
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
  console.log(position);
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
  const mmu_strains = await fetch(SPARQLIST_TOGOVAR_SPARQLIST + "/api/mouse_strain?strain_id=all", options).then(d => d.json());
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
  console.log(mog_body);
  const mogp = await fetch("https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?" + mog_body).then(d=>d.text());
  console.log(mogp);
  const mogp_res = mogp.split(/\n/);
  let pos_list = mogp_res[0].split(/\t/);
  let var_list = mogp_res[1].split(/\t/);
  console.log(pos_list);
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
          // console.log("strain_name="+ strain_name + ",source=" + JSON.stringify(strain2id))
	  mouse_ids.push(strain2id[strain_name]);
          if(strain2id[strain_name]?.source){
            mouse_strains.push("<a href=" + strain2id[strain_name].source + ">" + strain_name + "</a>");
          }else{
            mouse_strains.push(strain_name);
          }
        }

        const [interpretation, condition, condition_link] = get_clinsig(rs, clinvar)
  
        r.push({
          tgv_id: tgv_id == "-" ? "" : tgv_id,
          tgv_link: tgv_id == "-" ? "" : "/variant/" + tgv_id,
          rs: rs == "-" ? "" : rs,
          rs_link: rs == "-" ? "" : "https://www.ncbi.nlm.nih.gov/snp/" + rs,
          allele_grch38: chr + ":" + hsa_pos + "-" + hsa_ref + "-" + hsa_alt,
          consequence: consequence.replaceAll(",", "<br/>"),
          clinsig: interpretation ? interpretation : "",
          condition: condition ? condition : "",
          condition_link: condition_link ? condition_link : "",
          allele_grcm39: mmu_chr + ":" + mmu_pos + "-" + mmu_ref + "-" + mmu_alt,
          ref_grcm39: mmu_ref,
          alt_grcm39: mmu_alt,
          alt_match: alt_match,
          mouse_strains: mouse_strains.join("<br/>"),
          mogplus_url: "https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?strainNoSlct=refGenome&strainNoSlct="
          + mouse_ids.join("&strainNoSlct=").replace(/\//g, "_") + "&chrName=" + mmu_chr + mmu_var_pos[mmu_pos].region
          + "&seqType=genome&chrName=5&geneNameSearchText=&index=submit&presentType=disp"
        })
      }
    }
  });
  /*if (r[0]) r.unshift(["tgv_id", "rs", "chr", "position_grch38", "ref", "alt", "symbol", "transcript_id", 
                       "consequence", "sift_qualitative_prediction", "sift_score", "polyphen2_qualitative_prediction",
                       "polyphen2_score", "alphamissense_pathogenicity", "alphamissense_score", "chr_grcm39", "position_grcm39", "ref_grcm39", "alt_grcm39", "alt_match", "mouse_strains"].join("\t")); */
  //return r.join("\n");
  return r;
}
```
