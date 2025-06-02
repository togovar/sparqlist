# Search for the counterpart human variant by GRCm39 coordinates

* LintOver from GRCm39(mm39) to GRCh38(hg38) (千葉さん API (UCSC chain file))
* mouse->human: Search variant by TogoVar API
* reference genome の ref / alt を返す
  * "strand" は source(+) に対する target のマッピング向き + / - を意味するが、"position" は target の reference 鎖での数値

## Parameters

* `chr`
  * default: 11
  * example: 11, 11
* `pos`
  * default: 69479987
  * example: 69479987, 69471702
* `ref`
  * default: G
  * example: G, G
* `alt`
  * default: T
  * example: T (alt_match=true), T (alt_match=false)
* `mogplus_ver` : version for togov -> mogplus
  * default: mogplus21
  * example: mogplus3
* `ignore_alt_mismatch`
  * default: true
  * example: false

## `sequence`
```javascript
async ({chr, pos, ref, alt, mogplus_ver, ignore_alt_mismatch, SPARQLIST_TOGOVAR_APP})=>{
  // convert strand
  const conv_nt = (strand, nt) => {
    if (strand == "+") return nt;
    if (nt == "A") return "T";
    if (nt == "T") return "A";
    if (nt == "C") return "G";
    if (nt == "G") return "C";
  }
  
  // GRCh38 <=> GRCm39 by Chiba-san API
  let liftover_api = "https://orth.dbcls.jp/cgi-bin/liftOver/";
  let map = "mm39ToHg38";
  let position = "/chr" + chr + ":" + pos + "-" + pos;
  let text = await fetch(liftover_api + map + position).then(d=>d.text());
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
      break;
    }
  }
  
  // TogoVar
  const togovar_api = SPARQLIST_TOGOVAR_APP + "/api/search/variant";
  let options = {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/json'
    }
  };
  options.body = JSON.stringify({
    offset: 0,
    query: {
      and: [
        {
          location: {
            chromosome: counter_chr,
            position: counter_pos
          }
        },
        {
          type: {
            relation: "eq",
            terms: ["snv"]
          }
        }
      ]
    }
  });

  const json = await fetch(togovar_api, options).then(d=>d.json());

  let r = [];
  for (const d of json.data) {
    if (alt == conv_nt(strand, d.alternate) || ignore_alt_mismatch == "true"){
      r.push({
        togovar_url: "https://grch38.togovar.org/variant/" 
          + d.chromosome + "-" + d.position + "-" + d.reference + "-" + d.alternate,
        target: "GRCh38",
        chr: d.chromosome,
        pos: d.position,
        ref: d.reference,
        alt: d.alternate,
        strand: strand,
        alt_match: alt == conv_nt(strand, d.alternate) ? true : false,
        data: d
      });
    }
  }

  return r;
}
```
