# Convert human variants to mouse genome positions from mouse genome range

## Parameters

* `mouse_range`
  * example: chr5:121702449-121731689
* `togovar_stg` : staging
  * default: grch38
  * example: grch38
* `mogplus_ver` : version
  * default: mogplus21
  * example: mogplus3
* `mogplus_strain_api`
  * default: https://grch38.togovar.org/sparqlist/api/mouse_strain

## `range`
```javascript
async ({mouse_range}) => {
  // 千葉さん API (UCSC chain file)
  // mouse => GRCh38
  const api = "https://orth.dbcls.jp/cgi-bin/liftOver/mm39ToHg38/";
  const liftover = await fetch(api + mouse_range).then(d => d.text());
  const list = liftover.split(/\n/);
  let hsa_chr, strand, hsa_pos1, hsa_pos2;
  for (const l of list) {
    if (l.match(/^mm/)) {
      [,,,,,hsa_chr, strand, hsa_pos1] = l.split(/\s/);
      break;
    }
  }
  [,,,,,,, hsa_pos2] = list[list.length - 2].split(/\s/);
  const begin = strand == "+" ? hsa_pos1 : hsa_pos2;
  const end = strand == "+" ? hsa_pos2 : hsa_pos1;
  return {
    chr: hsa_chr,
    begin: begin,
    end: end
  };
}
```

## `sequence`
```javascript
async ({togovar_stg, mogplus_ver, mogplus_strain_api, range})=>{
  if (! range.chr) return "Gneome position not found";
  const chr    = range.chr.replace("chr", "");
  const begin  = range.begin;
  const end    = range.end;

  // make strain list
  const strain2id = await fetch(mogplus_strain_api + "?strain=all", {
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


  // 千葉さん API (UCSC chain file)
  // GRCh38 => GRCm39(mm39)
  api = "https://orth.dbcls.jp/cgi-bin/liftOver/";
  position = "/chr" + position;
  let mmu_chr, mmu_start, mmu_end, mmu_strand;
  const map = "hg38ToMm39";
  console.log(api + map + position);
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
  const mmu_strains = await fetch(mogplus_strain_api + "?strain_id=all", options).then(d => d.json());
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
  console.log("https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?" + mog_body);
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
        r.push({
          tgv_id: tgv_id,
          rs: rs,
          chr: chr,
          position_grch38: parseInt(hsa_pos),
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
          mouse_strains: mouse_strains.join("<br/>"),
          mogplus_url: "https://molossinus.brc.riken.jp/" + mogplus_ver+ "/variantTable/?strainNoSlct=refGenome&strainNoSlct="
          + mouse_ids.join("&strainNoSlct=").replace(/\//g, "_") + "&chrName=" + mmu_chr + mmu_var_pos[mmu_pos].region
          + "&seqType=genome&chrName=5&geneNameSearchText=&index=submit&presentType=disp"
        });
      }
    }
  });
  return r;
}
```
