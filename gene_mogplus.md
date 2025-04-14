# Convert human vriants to mouse genome positions per gene

## Parameters

* `hgnc` : hgnc
  * default: 404
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
  VALUES ?hgnc_uri { hgnc:{{hgnc}} }
  ?hgnc_uri dct:identifier ?hgnc ;
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

## `sequence`
```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, hgnc, mogplus_ver, symbol, range})=>{
  if (! symbol.results.bindings[0]) return "Symbol not found";
  if (! range.results.bindings[0]) return "Gneome position not found";
  const chr    = range.results.bindings[0].chr.value.replace("http://identifiers.org/hco/", "").replace("/GRCh38","");
  const begin  = range.results.bindings[0].begin.value;
  const end    = range.results.bindings[0].end.value;  
  
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

  /*
  // 千葉さん API (UCSC chain file)
  api = "https://orth.dbcls.jp/cgi-bin/liftOver/";
  position = "/chr" + position;
  let mmu38_chr, mmu39_chr, mmu39_start, mmu39_end, mmu39_strand;
  // GRCh38 => GRCm38(mm10) by 
  let map = "hg38ToMm10";
  const liftover = await fetch(api + map + position).then(d=>d.text());
  //console.log(liftover);
  let list = liftover.split(/\n/);
  let hsa2mmu38 = {};
  for (const l of list) {
    if (l.match(/^NOT_FOUND/)) return "Lifover position not founr";
    if (l.match(/^hg/)) {
      let [,,,hsa_pos,,mmu_chr_t, strand, mmu_pos] = l.split(/\s/);
      if (!mmu38_chr) mmu38_chr = mmu_chr_t.replace(/chr/, "");
      //if (!mmu_strand) mmu_strand = strand;
      mmu_pos = parseInt(mmu_pos);
      //if (!mmu_start) mmu_start = mmu_pos;
      //mmu_end = mmu_pos;
      hsa2mmu38[hsa_pos] = mmu_pos;
    }
  } */
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
  // console.log(mog_body);
  const mogp = await fetch("https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?" + mog_body).then(d=>d.text());
  console.log(mogp);
  const mogp_res = mogp.split(/\n/);
  let pos_list = mogp_res[0].split(/\t/);
  let var_list = mogp_res[1].split(/\t/);
  console.log(pos_list);
  if (!pos_list[1]) return "Mog+ variant not found";

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
    const [tgv_id, rs, chr , hsa_pos, hsa_ref, hsa_alt, symbol, transcript_id, consequence, sift_qualitative_prediction, 
           sift_score, polyphen2_qualitative_prediction, polyphen2_score, alphamissense_pathogenicity, alphamissense_score] = d.split(/\t/);
    const mmu_pos = hsa2mmu[hsa_pos];
    let mmu_ref;
    if (mmu_var_pos[mmu_pos]?.ref) mmu_ref = conv_nt(mmu_strand, mmu_var_pos[mmu_pos].ref);
    if (mmu_ref == hsa_ref && mmu_var_pos[mmu_pos]?.alt[0] && hsa_ref.length == 1 && hsa_alt.length == 1) {
      for (const mmu_alt of mmu_var_pos[mmu_pos].alt) {
        let alt_match = false;
        if (hsa_alt == conv_nt(mmu_strand, mmu_alt)) alt_match = true;
       // r.push(d + "\t" + [mmu_chr, mmu_pos, mmu_var_pos[mmu_pos].ref, mmu_alt, alt_match, mmu_var_pos[mmu_pos].strain[mmu_alt].join(",")].join("\t"));
	    r.push({
          tgv_id: tgv_id, 
          rs: rs, 
          chr: chr,
          position_grch38: hsa_pos,
          ref: hsa_ref,
          alt: hsa_alt,
          symbol: symbol,
          transcript_id: transcript_id,
          consequence: consequence,
          sift_qualitative_prediction: sift_qualitative_prediction, 
          sift_score: sift_score,
          polyphen2_qualitative_prediction: polyphen2_qualitative_prediction,
          polyphen2_score: polyphen2_score,
          alphamissense_pathogenicity: alphamissense_pathogenicity,
          alphamissense_score: alphamissense_score,
          chr_grcm39: mmu_chr,
          position_grcm39: mmu_pos,
          ref_grcm39: mmu_var_pos[mmu_pos].ref,
          alt_grcm39: mmu_alt,
          alt_match: alt_match,
          mouse_strains: mmu_var_pos[mmu_pos].strain[mmu_alt].join(",") /* ,
          mogplus_url: "https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?strainNoSlct=refGenome&strainNoSlct="
          + mmu_var_pos[mmu_pos].strain[mmu_alt].join("&strainNoSlct=").replace(/\//g, "_") + "&chrName=" + mmu_chr + mmu_var_pos[mmu_pos].region
          + "&seqType=genome&chrName=5&geneNameSearchText=&index=submit&presentType=disp" */
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
