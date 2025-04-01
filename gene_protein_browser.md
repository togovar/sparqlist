# Gene / Protein browser

## Parameters

* `hgnc_id`
  * default: 2433
* `jpost_endpoint`
  * default: https://tools.jpostdb.org/proxy/sparql
* `glycosmos_endpoint`
  * default: https://ts.glycosmos.org/sparql

## Endpoint
https://rdfportal.org/sib/sparql

## `hgnc2other`
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
           rdfs:seeAlso ?enst ;
           core:sequence [
             a core:Simple_Sequence ;
             rdf:value ?seq 
           ] .
  ?enst core:translatedTo ?ensp .
  OPTIONAL {
    ?enst rdfs:seeAlso ?isoform .
    ?isoform a core:Simple_Sequence .
  }
}
```

## `id`
```javascript
({hgnc2other}) => {
  const uniprot = hgnc2other.results.bindings[0].uniprot.value.replace("http://purl.uniprot.org/uniprot/", "");
  const seq = hgnc2other.results.bindings[0].seq.value;
  // core:Simple_Sequence
  for (const d of hgnc2other.results.bindings) {
    if (d.isoform && d.isoform.value) {
      return {uniprot: uniprot, seq: seq, ensp: d.ensp.value.replace(/.+ensembl\.protein\//, "").replace(/\.\d+$/, "")};
    }
  }
  // mane
  let mane;
  for (const d of hgnc2other.results.bindings) {
    if (d.enst.value.match(/mane-select/)) {
      mane = d.enst.value.replace(/.+mane-select\//, "").replace(/\.\d+$/, "");
      break;
    }
  }
  for (const d of hgnc2other.results.bindings) {
    if (!d.enst.value.match(/mane-select/) && d.enst.value.match(mane)) {
      return {uniprot: uniprot, seq: seq, ensp: d.ensp.value.replace(/.+ensembl\.protein\//, "").replace(/\.\d+$/, "")};
    }
  }
  for (const d of hgnc2other.results.bindings) {
    if (!d.enst.value.match(/mane-select/)) {
      return {uniprot: uniprot, seq: seq, ensp: d.ensp.value.replace(/.+ensembl\.protein\//, "").replace(/\.\d+$/, "")};
    }
  }
  return {uniprot: uniprot, seq: seq, ensp: false};
}
```

## `uniprot_ptm`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX up: <http://purl.uniprot.org/uniprot/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX core: <http://purl.uniprot.org/core/>
SELECT DISTINCT ?label ?position ?an_type
WHERE {
  VALUES ?an_type { core:Modified_Residue_Annotation core:Glycosylation_Annotation }
  up:{{id.uniprot}} core:annotation ?an .
  ?an a	?an_type ; 
      rdfs:comment ?label ;
      core:range/faldo:begin/faldo:position ?position .
}
ORDER BY ?position
```

## `uniprot_substitution`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX up: <http://purl.uniprot.org/uniprot/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX core: <http://purl.uniprot.org/core/>
SELECT DISTINCT ?substitution ?begin ?end ?dbsnp ?comment
WHERE {
  up:{{id.uniprot}} core:annotation ?an .
  ?an a	core:Natural_Variant_Annotation ;
      core:substitution ?substitution ;
      core:range [ 
        faldo:begin/faldo:position ?begin ;
        faldo:end/faldo:position ?end 
      ] .
  OPTIONAL {
    ?an rdfs:seeAlso ?dbsnp .
  }
  OPTIONAL {
    ?an rdfs:comment ?comment .
  }
}
ORDER BY ?position
```

## Endpoint
{{jpost_endpoint}}

## `psm`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX up: <http://purl.uniprot.org/uniprot/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX jpo: <http://rdf.jpostdb.org/ontology/jpost.owl#>
PREFIX : <http://rdf.jpostdb.org/entry/>
SELECT ?seq ?begin ?end (COUNT (DISTINCT ?psm) AS ?count)
WHERE {
  ?protein jpo:hasDatabaseSequence up:{{id.uniprot}} ;
           jpo:hasPeptideEvidence ?pepevi .
  ?pepevi faldo:location/faldo:begin/faldo:position ?begin ;
          faldo:location/faldo:end/faldo:position ?end ;
          jpo:hasPeptide ?pep .
  ?pep jpo:hasSequence [ a obo:MS_1001344 ;
                           rdf:value ?seq ] ;
       jpo:hasPsm ?psm .
}
ORDER BY ?begin ?end
```

## `phospho`
```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX up: <http://purl.uniprot.org/uniprot/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX unimod: <http://www.unimod.org/obo/unimod.obo#>
PREFIX jpo: <http://rdf.jpostdb.org/ontology/jpost.owl#>
PREFIX : <http://rdf.jpostdb.org/entry/>
SELECT ?mods ?mod_label ?site ?pos (COUNT (DISTINCT ?psm) AS ?count)
WHERE {
  VALUES ?mods { unimod:UNIMOD_21 }  ## phospho id
  ?mods rdfs:label ?mod_label .
  ?protein jpo:hasDatabaseSequence up:{{id.uniprot}} ;
           jpo:hasPeptideEvidence ?pepevi .
  ?pepevi faldo:location [faldo:begin/faldo:position ?begin ] ;
          jpo:hasPeptide/jpo:hasPsm ?psm .
  ?psm jpo:hasModification ?blank .
  ?blank a ?mods ;
         faldo:location/faldo:position ?position ;
         jpo:modificationSite ?site .
  BIND (?position + ?begin -1 AS ?pos)
}
ORDER BY ?pos
```

## Endpoint
{{glycosmos_endpoint}}

## `glyco`
```sparql
PREFIX gco: <http://purl.jp/bio/12/glyco/conjugate#>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX sio: <http://semanticscience.org/resource/>
SELECT DISTINCT ?pos (GROUP_CONCAT(DISTINCT ?gtc_id ; separator = ",") AS ?gtcs) (GROUP_CONCAT(DISTINCT ?db ; separator = ",") AS ?dbs)
FROM <http://rdf.glycosmos.org/glycoprotein>
FROM <http://rdf.glycosmos.org/glycans/seq>
FROM <http://rdf.glycosmos.org/glycans/subsumption>
WHERE{
  VALUES ?protein {<http://glycosmos.org/glycoprotein/{{id.uniprot}}>}
  ?protein gco:glycosylated_at ?glycosylated .
  ?glycosylated sio:SIO_000772 ?db ;
                faldo:location / faldo:position ?pos .
  OPTIONAL{
    ?glycosylated gco:has_saccharide ?gtc .
    BIND(STRAFTER(STR(?gtc), "http://rdf.glycoinfo.org/glycan/") AS ?gtc_id)
  }
}
ORDER BY ?pos
```

## `return`

```javascript
async ({SPARQLIST_TOGOVAR_APP, hgnc_id, id, uniprot_ptm, uniprot_substitution, psm, phospho, glyco}) => {
  const a3to1 = {Gly: "G", Ala: "A", Leu: "L", Met: "M", Phe: "F", Trp: "W", Lys: "K", 
              Gln: "Q", Glu: "E", Ser: "S", Pro: "P", Val: "V", Ile: "I", Cys: "C",
              Tyr: "Y", His: "H", Arg: "R", Asn: "N", Asp: "D", Thr: "T", Ter: "*",
              del: "-", any: "?"};
  const a1to3 = {G: "Gly", A: "Ala", L: "Leu", M: "Met", F: "Phe", W: "Trp", K: "Lys", 
              Q: "Qln", E: "Glu", S: "Ser", P: "Pro", V: "Val", I: "lle", C: "Cys",
              Y: "Tyr", H: "His", R: "Arg", N: "Ans", D: "Asp", T: "Thr"};
  
  let res = {seq: id.seq};
  let list = [];
  
  // uniprot
  let uniprot_data = [];
  let y_flag = [];
  list = uniprot_ptm.results.bindings;
  for(const d of list){
    let y, type, symbol;
    if (d.label.value.match(/^Phospho/)) {
      y_flag[0] = true;
      y = 1;
      symbol = "p";
      if (d.label.value.match(/^Phosphoserin/)) { type = "ptm_p_S";}
      else if (d.label.value.match(/^Phosphothreonine/)) { type = "ptm_p_T";}
      else if (d.label.value.match(/^Phosphotyrosine/)) { type = "ptm_p_Y";}
    } else if(d.an_type.value.match(/Glycosylation_Annotation/)){ 
      y_flag[1] = true;
      y = 2;
      symbol = "g";
      type = "ptm_g"; 
    } else {
      y_flag[2] = true;
      y = 3;
      if (d.label.value.match(/[- ]acetyl/)) {symbol = "a"; type = "ptm_a";}
      else if (d.label.value.match(/[- ]methyl/) 
         || d.label.value.match(/[- ]dimethyl/) 
         || d.label.value.match(/[- ]trimethyl/)) { symbol = "m"; type = "ptm_m";}
      else if (d.label.value.match(/[- ]carboxyl/)) { symbol = "c"; type = "ptm_c";}
      else if (d.label.value.match(/[- ]succinyl/)) { symbol = "s"; type = "ptm_s";} 
      else {symbol = "*"; label=""; type = "ptm_other";}
    }
    uniprot_data.push({
      label: d.label.value,
      position: parseInt(d.position.value),
      highlight_range: 1,
      symbol: symbol,
      link: "https://www.uniprot.org/uniprot/" + id.uniprot + "/entry#ptm_processing",
      html: "<div class='ref-alt ml-26'>" + d.position.value + "</div><span class='pos-26'>" + d.label.value + "</span>",
      type: type,
      y: y
    });      
  }
  list = uniprot_substitution.results.bindings;
  for (const d of list) {
    y_flag[3] = true;
    const begin = parseInt(d.begin.value);
    const end = parseInt(d.end.value);
    const alt = d.substitution.value;
    let symbol = alt;
    if (d.substitution.value.length != 1) symbol = "-";
    const ref = id.seq.slice(begin - 1, end);
    let ref_tmp = ref;
    let alt_tmp = alt;
    if (ref_tmp.length == 1) ref_tmp = a1to3[ref_tmp];
    else if (!ref_tmp) ref_tmp = "";
    else ref_tmp = ref_tmp.length + "aa";
    if (alt_tmp.length == 1) alt_tmp = a1to3[alt_tmp];
    else if (!alt_tmp) alt_tmp = "";
    else alt_tmp = alt_tmp.length + "aa";
    let dbsnp = "";
    let comment = [];
    if (d.dbsnp && d.dbsnp.value.match(/dbsnp\/rs\d+/)) dbsnp = d.dbsnp.value.match(/dbsnp\/(rs\d+)/)[1];
    if (d.comment && d.comment.value) comment = "<span class='pos-26'>" + d.comment.value + "</span>";
    uniprot_data.push({
      dbsnp: dbsnp,
      label: ref + " > " + alt + " Natural variant",
      position: begin,
      highlight_range: end - begin + 1,
      ref: ref,
      alt: alt,
      site: id.seq.slice(begin - 1, end),
      symbol: symbol,
      link: "https://www.uniprot.org/uniprot/" + id.uniprot + "/entry#disease_variants",
      html: "<div class='ref-alt ml-26'>" + begin + " <span class='ref'>" + ref_tmp + "</span><span class='arrow'></span><span class='alt'>" + alt_tmp + "</span> " + dbsnp + "</div>" + comment,
      type: "sig_X",
      y: 4
    });
  }
  for (const [i, d] of uniprot_data.entries()) {
    let delta = 0;
    for (let j = 0; j < d.y; j++) {
      if (!y_flag[j]) delta++;
    }
    if (delta > 0) uniprot_data[i].y -= delta;
  }
  res.uniprot = uniprot_data;
  
  // phospho
  list = psm.results.bindings;
  let posFreq = [];
  let phospho_data = [];
  for (const d of list) {
    for (let i = parseInt(d.begin.value); i <= parseInt(d.end.value); i++){
      if(!posFreq[i]) posFreq[i] = 0;
      posFreq[i] += parseInt(d.count.value);
    }
  }
  list = phospho.results.bindings;
  for (const d of list) {
    let label = "Phosphoserin"; // S
    if (d.site.value == "T") label = "Phosphothreonine";
    else if (d.site.value == "Y") label = "Phosphotyrosine";
    const pos = parseInt(d.pos.value);
    phospho_data.push({
      label: label,
      position: pos,
      highlight_range: 1,
      position_count: posFreq[pos],
      count: parseInt(d.count.value),
      site: d.site.value,
      link: "https://globe.jpostdb.org/protein.php?id=" + id.uniprot,
      html: "<div class='ref-alt ml-26'>" + pos + "</div><span class='pos-26'>" + label + "<br>Frequency: " + parseInt(d.count.value) + " / " + posFreq[pos] + "</span>",
      type: "ptm_p_" + d.site.value
    });
  }
  res.phospho = phospho_data;

  // glycan
  list = glyco.results.bindings;
  let glycan_data = [];
  for (const d of list) {
    const pos = parseInt(d.pos.value);
    let label = "Glycosylation";
    const glycans = d.gtcs.value.split(/,/);
    let glycans_t = "";
    if (glycans[0]?.match(/^G/)) {
      glycans_t = "<br>Glycan: " + glycans[0] + "<br><img class='gtc_png' src='https://image.glycosmos.org/snfg/png/" + glycans[0] + "'>";
      if (glycans.length > 1) glycans_t += "<br>... +" + (glycans.length - 1) + " glycans";
    }
    let highlight_range = 1;
    if (id.seq.slice(pos - 1, pos + 2).match(/N[A-OQ-Z][ST]/)) {
      label += "<br>Sequon: <span class='sequon'>" + id.seq.slice(pos - 1, pos + 2).match(/(N[A-OQ-Z][ST])/)[1] + "</span>";
      highlight_range = 3;
    }
    glycan_data.push({
	  label: "Glycosylation",
	  position: pos,
      highlight_range: highlight_range,
	  site: id.seq.slice(pos - 1, pos),
	  symbol: "g",
      link: "https://glycosmos.org/glycoproteins/" + id.uniprot + "#GlycosylationSites",
      html: "<div class='ref-alt ml-26'>" + pos + " </div><span class='pos-26'>" + label + glycans_t + "</span>",
	  type: "ptm_g",
      y: 1
    });
  }
  res.glycan = glycan_data;

  // variant
  const cln_sig = {P: 1, LP: 2, PLP: 3, LPLP: 4, ERA: 5, LRA: 6, URA: 7, US: 8, LB:9, B: 10,
                   CI: 11, DR: 12, CS: 13, RF: 14, A: 15, PR: 16, AF: 17, O: 18, NP: 19, AN: 20, X: 99};
  const tgv_bdy = `{"offset":#offset,"limit":#limit,"query":{"and":[{"gene":{"relation":"eq","terms":[#hgncid]}},
    {"consequence":{"relation":"eq","terms":["missense_variant","frameshift_variant"]}}]}}`;
  let tgv_opt = {method: 'POST', headers: {'Accept': 'application/json', 'Content-Type': 'application/json'}}
  
  const togovar_api = SPARQLIST_TOGOVAR_APP + "/api/search/variant";
  let filtered = false;
  const limit = 500;
  let offset = 0;
  let count = 0;
  let stat_off = "";
  let variant_data= [];
  while (!filtered || filtered > count) {
    tgv_opt.body = tgv_bdy.replace(/#hgncid/, hgnc_id).replace(/#offset/, offset).replace(/#limit/, limit);
    const togovar = await fetch(togovar_api + stat_off, tgv_opt).then(res => res.json());
    if (!filtered) filtered = togovar.statistics.filtered;
    if (filtered == 0) break;
    if (togovar.data) {
	  for (const v of togovar.data) {
        if (!v.id) continue;
        let ref, alt, pos, hgvs_p;
        for (const t of v.transcripts) {
          if (t.hgvs_p?.match(id.ensp)) {
            if (t.hgvs_p.match(/:p\.[A-Z][a-z]{2}\d+.+/)) {
              [, hgvs_p, ref, pos, tmp] = t.hgvs_p.match(/:p\.(([A-Z][a-z]{2})(\d+)(.+))/);
              if (tmp.match(/^[A-Z][a-z]{2}/)) alt = tmp.match(/^([A-Z][a-z]{2})/)[1];
              else if(tmp.match(/^_[A-Z][a-z]{2}/)) alt = array[3].match(/^_([A-Z][a-z]{2})/)[1];
              else if(tmp.match(/^_.+ins[A-Z][a-z]{2}/)) alt = tmp.match(/^_.+ins([A-Z][a-z]{2})/)[1];
              else if(tmp.match(/^delins[A-Z][a-z]{2}/)) alt = tmp.match(/^delins([A-Z][a-z]{2})/)[1];
              else if(tmp.match(/del$/)) alt = "del";
              else if(tmp.match(/^dup/)) alt = ref;
              else if(tmp == "?") alt = "any";
              break;
            }
          }
        }
        if (!ref || !alt) continue;

        let min_sig = "X";
        let sigs = [];
        if (v.significance) {
          for (const s of v.significance) {
            if (!s.interpretations[0]) continue;
            if (cln_sig[min_sig] > cln_sig[s.interpretations[0]]) min_sig = s.interpretations[0];
            const sig_ico = " <span class='clinical-significance ml-26' data-sign='" + s.interpretations[0] + "'></span> ";
            if (s.conditions[0]) {
              for (const c of s.conditions) {
	        sigs.push(sig_ico + c.name);
	      }
            } else {
              sigs.push(sig_ico + "others");
            }
          }
        }

        let html = "<div class='ref-alt ml-26'>" + pos + " <span class='ref'>" + ref + "</span><span class='arrow'></span><span class='alt'>" + alt + "</span> " + v.id + "</div>";
        if (sigs[0]) html += sigs.join("<br>");
       
        variant_data.push({
          tgvid: v.id,
          position: parseInt(pos),
          highlight_range: 1,
          ref: a3to1[ref], 
          alt: a3to1[alt],
          hgvs_p: hgvs_p,
          symbol: a3to1[alt],
          link: SPARQLIST_TOGOVAR_APP + "/variant/" + v.id,
          html: html,
          type: "sig_" + min_sig,
          sig_level: cln_sig[min_sig]
        });
      }
    }
    offset = '["' + togovar.data[togovar.data.length - 1].chromosome.replace("X", "23").replace("Y", "24").replace("MT", "25") + '","' + togovar.data[togovar.data.length - 1].vcf.position + '","' + togovar.data[togovar.data.length - 1].vcf.reference + '","' + togovar.data[togovar.data.length - 1].vcf.alternate + '"]';
    count += limit;
    stat_off = "?stat=0";
  }
  variant_data.sort(function(a,b){
	if( a.position < b.position ) return -1;
	if( a.position > b.position ) return 1;
	if( a.sig_level < b.sig_level ) return 1;
	if( a.sig_level > b.sig_level ) return -1; 
	return 0;
  });
  let pre = 0;
  let y = 1;
  let max_y = 0;
  for (let i = 0; i < variant_data.length; i++) {
    if (pre == variant_data[i].position) {
      y++;
      if (y > max_y) max_y = y;
    } else {
      y = 1;
    }
    pre = variant_data[i].position;
    variant_data[i].y = y;
  }
  for (let i = 0; i < variant_data.length; i++) {
    variant_data[i].y = max_y - variant_data[i].y + 1;
  }
  res.variant = variant_data;

  return res;
};
```
