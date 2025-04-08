# Search the counterpart variant between human and mouse 

* 双方向変換
* LinfOver between GRCh38(hg38) and GRCm39(mm39) (千葉さん API (UCSC chain file))
* mouse->human: Search variant by TogoVar API
* human->mouse: Search variant and strains by MoG+ API
* refeerenc genome の ref / alt を返す
  * "strand" は source(+) に対する target のマッピング向き + / - を意味するが、"position" は target の reference 鎖での数値

## Parameters

* `source`
  * default: GRCh38
  * example: GRCm39
* `chr`
  * default: 12
  * example: 5
* `pos`
  * default: 111781972
  * example: 121724761
* `ref`
  * default: G
  * example: C
* `alt`
  * default: A
  * example: T
* `mogplus_ver` : version for togov -> mogp
  * default: mogplus21
  * example: mogplus3

## `sequence`
```javascript
async ({source, chr, pos, ref, alt, mogplus_ver, SPARQLIST_TOGOVAR_APP})=>{
  let s = Date.now();
  // convert strand
  const conv_nt = (strand, nt) => {
    if (strand == "+") return nt;
    if (nt == "A") return "T";
    if (nt == "T") return "A";
    if (nt == "C") return "G";
    if (nt == "G") return "C";
  }
  
  // GRCh38 <=> GRCm39 by Chiba-san API
  let api = "https://orth.dbcls.jp/cgi-bin/liftOver/";
  let map = "hg38ToMm39"
  let position = "/chr" + chr + ":" + pos + "-" + pos;
  if (source == "GRCm39") map = "mm39ToHg38"
  let text = await fetch(api + map + position).then(d=>d.text());
  const list = text.split(/\n/);
  let counter_chr, counter_pos, strand;
  let chr_len, chain_start, chain_end;
  for (const l of list) {
    if (l.match(/^NOT_FOUND/)) return {error: "Not found lifover position"};
    if (l.match(/^chain/)) [,,,,,,,,chr_len,,chain_start,chain_end] = l.split(/\s/);
    if (l.match(/^hg/) || l.match(/^mm/)) {
      [,,,,,counter_chr, strand ,counter_pos] = l.split(/\s/);
      counter_chr = counter_chr.replace(/chr/, "");
      counter_pos = parseInt(counter_pos);
      chr_len = parseInt(chr_len);
      chain_start = parseInt(chain_start);
      chain_end = parseInt(chain_end);
     // if (strand == "-") counter_pos = chr_len + counter_pos - chain_start - chain_end; // old API
      break;
    }
  }
  console.log([counter_chr, strand ,counter_pos, chr_len]);
  console.log(Date.now() - s);
  
  if (source == "GRCm39") {
    // TogoVar
    const api = SPARQLIST_TOGOVAR_APP + "/api/search/variant";
    let options = {
      method: 'POST',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/json'
      }
    };
    options.body = '{"offset":0,"query":{"and":[{"location":{"chromosome":"' + counter_chr + '","position":' + counter_pos + '}},{"type":{"relation":"eq","terms":["snv"]}}]}}';
    const json = await fetch(api, options).then(d=>d.json());
    for (const d of json.data) {
      if (ref == conv_nt(strand, d.reference) && alt == conv_nt(strand, d.alternate)) {
        console.log(Date.now() - s);
        return {target: "GRCh38", chr: counter_chr, pos: counter_pos, ref: d.reference, alt: d.alternate, strand: strand, data: d};
      }
    }
    return {error: "No counterpart"};
  } else {
    // MoG+ GRCm39 variants
    let strains = [];
    let ver_strain = "strainNoSlct=msmv4_sq&strainNoSlct=jf1v3&strainNoSlct=kjrv1&strainNoSlct=swnv1&strainNoSlct=chdv1&"
     + "strainNoSlct=njlv1&strainNoSlct=blg2v1&strainNoSlct=hmiv1&strainNoSlct=bfmv1&strainNoSlct=pgn2v1&";
    if (mogplus_ver == "mogplus3") ver_strain = "strainNoSlct=msmb39gatk&strainNoSlct=jf1b39gatk&";
    const mog_body = "strainNoSlct=refGenome&" + ver_strain
     + "strainNoSlct=129P2_OlaHsd&strainNoSlct=129S1_SvImJ&strainNoSlct=129S5SvEvBrd&strainNoSlct=A_J&"
     + "strainNoSlct=AKR_J&strainNoSlct=B10.RIII&strainNoSlct=BALB_cByJ&strainNoSlct=BALB_cJ&strainNoSlct=BTBR_T%2B_Itpr3tf_J&"
     + "strainNoSlct=BUB_BnJ&strainNoSlct=C3H_HeH&strainNoSlct=C3H_HeJ&strainNoSlct=C57BL_10J&strainNoSlct=C57BL_10SnJ&"
     + "strainNoSlct=C57BL_6NJ&strainNoSlct=C57BR_cdJ&strainNoSlct=C57L_J&strainNoSlct=C58_J&strainNoSlct=CAST_EiJ&"
     + "strainNoSlct=CBA_J&strainNoSlct=CE_J&strainNoSlct=CZECHII_EiJ&strainNoSlct=DBA_1J&strainNoSlct=DBA_2J&strainNoSlct=FVB_NJ&"
     + "strainNoSlct=I_LnJ&strainNoSlct=JF1_MsJ&strainNoSlct=KK_HiJ&strainNoSlct=LEWES_EiJ&strainNoSlct=LG_J&strainNoSlct=LP_J&"
     + "strainNoSlct=MA_MyJ&strainNoSlct=MOLF_EiJ&strainNoSlct=NOD_ShiLtJ&strainNoSlct=NON_LtJ&strainNoSlct=NZB_B1NJ&"
     + "strainNoSlct=NZO_HlLtJ&strainNoSlct=NZW_LacJ&strainNoSlct=PL_J&strainNoSlct=PWK_PhJ&strainNoSlct=QSi3&strainNoSlct=QSi5&"
     + "strainNoSlct=RF_J&strainNoSlct=RIIIS_J&strainNoSlct=SEA_GnJ&strainNoSlct=SJL_J&strainNoSlct=SM_J&strainNoSlct=SPRET_EiJ&"
     + "strainNoSlct=ST_bJ&strainNoSlct=SWR_J&strainNoSlct=WSB_EiJ&strainNoSlct=ZALENDE_EiJ&"
     + "chrName=" + counter_chr + "&chrStart=" + counter_pos + "&chrEnd=" + counter_pos + "&"
     + "&seqType=genome&chrName=5&geneNameSearchText=&index=submit&presentType=dwnld";
    const mogp = await fetch("https://molossinus.brc.riken.jp/" + mogplus_ver + "/variantTable/?" + mog_body).then(d=>d.text());
    const mogp_res = mogp.split(/\n/);
    console.log("https://molossinus.brc.riken.jp/" + mogplus_ver + "/variantTable/?" + mog_body);
    let pos_list = mogp_res[0].split(/\t/);
    if (!pos_list[1]) return {error: "No cunterpart", "api": "https://molossinus.brc.riken.jp/" + mogplus_ver + "/variantTable/?" + mog_body};
    let counter_ref = mogp_res[1].split(/\t/)[1];
    let counter_alt = '';
    for (let i = 2; i < mogp_res.length; i++) {
      let line = mogp_res[i].split(/\t/);
      if (ref == conv_nt(strand, counter_ref) && alt == conv_nt(strand, line[1])) {
        counter_alt = line[1];
        strains.push(line[0]);
      }
    } 
    if (strains[0]) {
      console.log(Date.now() - s);
      return {target: "GRCm39", chr: counter_chr, pos: counter_pos, ref: counter_ref, alt: counter_alt, strand: strand, strains: strains};
    }
    return {error: "No counterpart"};
  }
}
```
