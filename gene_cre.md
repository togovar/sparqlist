# Gene to CREs in human and mouse

Successor to `togovar_cre`, which is left in place untouched.

* What changed from `togovar_cre`
  * CREs come from the **fanta.bio RDF on RDF Portal** (`togovar_cre` reads the
    JSON Lines grep API). That is v1.2.1 against the JSON's v1.2.0, it answers
    in about 200 ms instead of about 5 s, and it is the same version TogoCoord
    ingested, so a CRE the CRE list names is no longer missing from TogoCoord.
  * Every CRE of the gene is returned, **including those with no ChIP-Atlas
    antigen**. They cannot form a TF pair, but they hold variants.
  * Entry from **either species**: `hgnc`, `mgi` or `ncbigene`. The ortholog is
    resolved by the same NCBI Orthologs RDF endpoint in both directions, so the
    stanza no longer needs its own ortholog lookup.
  * All coordinates and liftOver come from **TogoCoord** instead of the
    `orth.dbcls.jp/cgi-bin/liftOver` CGI: CRE positions, the human <-> mouse
    correspondence, **and the mouse GRCm39 position** that MoG+ needs.
  * **Fixed**: the shared-TF count was mutating one shared map across the human
    loop, so a TF was credited to only the first human CRE it matched. See
    `return`.
  * Added: gene transcription start and strand per species, aligned base counts,
    and GRCm39 coordinates for every CRE.

## Parameters

* `hgnc`
  * default: 404
  * example: 404 (ALDH2), 10848 (SHH)
* `mgi` : MGI gene ID, when entering from mouse
  * example: 99600 (Aldh2)
* `ncbigene` : NCBI Gene ID of a human or mouse gene
  * example: 11669 (mouse Aldh2), 217 (human ALDH2)
* `togocoord`
  * default: https://togocoord.dbcls.jp/

## `gene`
```javascript
async ({hgnc, mgi, ncbigene}) => {
  // One NCBI Gene ID, whichever identifier came in, and which species it is.
  // The species is settled here so that the ortholog step can run the one
  // query that applies instead of a UNION that tries both ends.
  const togoid = async (ids, route) => {
    const url = "https://api.togoid.dbcls.jp/convert?route=" + encodeURIComponent(route)
      + "&report=target&format=json&ids=" + encodeURIComponent(ids);
    const json = await fetch(url).then(res => res.json());
    return json.results || [];
  };
  // is_human / is_mouse rather than one field, because the ortholog query picks
  // its half with a Handlebars conditional, which tests truthiness.
  const result = (ncbigene_id, species, entry) => ({
    ncbigene: ncbigene_id, species, entry,
    is_human: species == "human" || "", is_mouse: species == "mouse" || ""
  });
  if (mgi) {
    const id = String(mgi).replace(/^MGI:?/i, "");
    const [found] = await togoid(id, "mgi_gene,ncbigene");
    if (!found) throw new Error("No NCBI Gene for MGI:" + id);
    return result(found, "mouse", {type: "mgi", id: "MGI:" + id});
  }
  if (ncbigene) {
    const id = String(ncbigene).replace(/^ncbigene:?/i, "");
    // An NCBI Gene ID says nothing about its species, so it is asked whether it
    // has an HGNC ID; only human genes do.
    const [hgnc_id] = await togoid(id, "ncbigene,hgnc");
    return result(id, hgnc_id ? "human" : "mouse", {type: "ncbigene", id});
  }
  const id = String(hgnc).replace(/^HGNC:?/i, "");
  const [found] = await togoid(id, "hgnc,ncbigene");
  if (!found) throw new Error("No NCBI Gene for HGNC:" + id);
  return result(found, "human", {type: "hgnc", id: "HGNC:" + id});
}
```

## Endpoint

https://spang.dbcls.jp/sparql

## `orth`

One query, no UNION: the entry gene's species is known by now, so the
conditionals emit only the half that applies and the other half is never sent.
The trailing filter is for the case where neither fires -- an empty `WHERE {}`
returns one row with nothing bound rather than no rows at all.

```sparql
PREFIX orth: <http://purl.org/net/orth#>
PREFIX taxid: <http://identifiers.org/taxonomy/>
PREFIX ncbigene: <http://identifiers.org/ncbigene/>
PREFIX : <https://dbcls.github.io/ncbigene-rdf/ontology.ttl#>

SELECT DISTINCT ?human ?human_label ?mouse ?mouse_label
WHERE {
{{#if gene.is_human}}
  VALUES ?human { ncbigene:{{gene.ncbigene}} }
{{/if}}
{{#if gene.is_mouse}}
  VALUES ?mouse { ncbigene:{{gene.ncbigene}} }
{{/if}}
  ?human orth:hasOrtholog+ ?mouse ;
         :taxid taxid:9606 ;
         rdfs:label ?human_label .
  ?mouse :taxid taxid:10090 ;
         rdfs:label ?mouse_label .
}
```

## `cre`
```javascript
async ({gene, orth, togocoord}) => {
  const b = orth.results.bindings[0];
  if (!b) throw new Error("No human/mouse ortholog pair for ncbigene:" + gene.ncbigene
    + " (" + gene.species + ")");
  const id = (iri) => iri.replace(/.*\/ncbigene\//, "");
  const pair = {
    human: {ncbigene: id(b.human.value), symbol: b.human_label.value},
    mouse: {ncbigene: id(b.mouse.value), symbol: b.mouse_label.value}
  };
  const entry_species = gene.species;
  const base = (togocoord || "https://togocoord.dbcls.jp/").replace(/\/*$/, "/");

  //// ---- TogoCoord helpers -------------------------------------------------
  // A Location ID is read only as far as this API needs: the sequence, the
  // blocks and whether it is on the complementary strand.
  const parse_location = (loc) => {
    if (!loc) return null;
    const first = loc.indexOf(":");
    const second = loc.indexOf(":", first + 1);
    if (second < 0) return null;
    const body = loc.slice(second + 1);
    const blocks = [];
    for (const part of body.replace(/[a-z]+\(/g, "").replace(/[)<>]/g, "").split(",")) {
      const m = part.match(/^\s*(\d+)(?:(?:\.\.|\^|\.)(\d+))?\s*$/);
      if (!m) continue;
      const a = parseInt(m[1], 10);
      const c = m[2] === undefined ? a : parseInt(m[2], 10);
      blocks.push({start: Math.min(a, c), end: Math.max(a, c)});
    }
    if (!blocks.length) return null;
    return {
      sequence: loc.slice(0, second),
      complement: body.includes("complement("),
      blocks: blocks,
      low: Math.min(...blocks.map(x => x.start)),
      high: Math.max(...blocks.map(x => x.end)),
      aligned: blocks.reduce((s, x) => s + (x.end - x.start + 1), 0)
    };
  };

  // RefSeq chromosome accessions are numbered consecutively per assembly.
  const chrom_name = (sequence, taxon) => {
    const m = /NC_(\d+)\.\d+$/.exec(sequence || "");
    if (!m) return null;
    const n = parseInt(m[1], 10);
    if (taxon == 9606) return "chr" + (n <= 22 ? n : {23: "X", 24: "Y"}[n] || n);
    if (taxon == 10090) return "chr" + (n >= 67 && n <= 85 ? n - 66 : {86: "X", 87: "Y"}[n] || n);
    return null;
  };

  // The part of the input the first liftOver step could place, i.e. the input
  // minus the unmapped stretches at either end.
  const matched_range = (step, low, high) => {
    const un = step && step.unmapped ? parse_location(step.unmapped) : null;
    if (!un) return [low, high];
    const blocks = un.blocks.slice().sort((x, y) => x.start - y.start);
    let s = low, e = high;
    for (const x of blocks) if (x.start <= s && x.end >= s) s = x.end + 1;
    for (let i = blocks.length - 1; i >= 0; i--) {
      const x = blocks[i];
      if (x.end >= e && x.start <= e) e = x.start - 1;
    }
    return s <= e ? [s, e] : [low, high];
  };

  const convert = async (cre_ids, taxon, assembly) => {
    if (!cre_ids.length) return [];
    const res = await fetch(base + "v1/convert", {
      method: "POST",
      headers: {"Content-Type": "application/json"},
      body: JSON.stringify({
        locations: cre_ids.map(x => "fanta:" + x),
        to: "genome", taxon: String(taxon), assembly: assembly
      })
    });
    if (!res.ok) throw new Error("TogoCoord " + res.status);
    return (await res.json()).results || [];
  };

  // The gene itself, for its transcription start and strand.
  const gene_feature = async (sequence, from, to, ncbigene_id) => {
    if (!sequence) return null;
    const url = base + "v1/annotations?loc=" + encodeURIComponent(sequence + ":" + from + ".." + to);
    // Retried, then raised rather than swallowed. A result is cached per API
    // and parameters, so a single blip here used to be stored as a gene with no
    // transcription start and kept being served that way; failing is noisy but
    // self-correcting on the next request.
    let body = null, last = null;
    for (let attempt = 0; attempt < 3 && !body; attempt++) {
      if (attempt) await new Promise(r => setTimeout(r, 300 * attempt));
      try {
        const res = await fetch(url);
        if (!res.ok) throw new Error("HTTP " + res.status);
        body = await res.json();
      } catch (e) { last = e; }
    }
    if (!body) throw new Error("TogoCoord annotations failed for " + sequence + ":" + from + ".." + to
      + " (" + (last && last.message) + ")");
    for (const a of body.annotations || []) {
      if (a.type != "gene") continue;
      if (!((a.attributes || {}).Dbxref || []).includes("GeneID:" + ncbigene_id)) continue;
      // Mouse annotations come from GRCm39 while the CREs are GRCm38, so the
      // location that is on the track's own sequence is the one used.
      const here = [a.lifted, a.location].filter(Boolean).map(parse_location)
        .find(p => p && p.sequence == sequence);
      if (!here) continue;
      return {
        symbol: ((a.attributes || {}).gene || [])[0] || "",
        strand: here.complement ? "-" : "+",
        tss: here.complement ? here.high : here.low,
        gene_end: here.complement ? here.low : here.high,
        gene_start_pos: here.low,
        gene_end_pos: here.high,
        biotype: ((a.attributes || {}).gene_biotype || [])[0] || ""
      };
    }
    return null;
  };

  //// ---- fanta.bio (RDF, v1.2.1) -------------------------------------------
  // Replaces the JSON Lines grep API, which is v1.2.0 and takes about five
  // seconds per gene. Every CRE of the gene is returned, including the ones
  // with no ChIP-Atlas antigen: they cannot form a TF pair, but they carry
  // variants, which is the other half of what this page compares.
  const RDFPORTAL = "https://rdfportal.org/primary/sparql";
  const FANTA_GRAPH = "<http://rdfportal.org/dataset/fantabio>";
  const PREFIXES = `PREFIX sio: <http://semanticscience.org/resource/>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX fantao: <http://fanta.bio/ontology/>`;

  const ask = async (query) => {
    const res = await fetch(RDFPORTAL, {
      method: "POST",
      headers: {"Content-Type": "application/x-www-form-urlencoded", Accept: "application/sparql-results+json"},
      body: "query=" + encodeURIComponent(query)
    });
    if (!res.ok) throw new Error("RDF Portal " + res.status);
    return (await res.json()).results.bindings;
  };
  const val = (b, k) => (b[k] ? b[k].value : null);

  // A CRE may regulate several genes; they share one node, so asking for the
  // gene on that node finds every CRE the gene is listed on.
  const cre_query = (ncbigene) => `${PREFIXES}
SELECT DISTINCT ?cre_id ?cre_name ?chr ?begin ?end ?tss_distance
WHERE {
  GRAPH ${FANTA_GRAPH} {
    ?reg rdfs:seeAlso <http://identifiers.org/ncbigene/${ncbigene}> .
    ?cre sio:SIO_000395 ?reg ;
         dct:identifier ?cre_id ;
         rdfs:label ?cre_name ;
         faldo:location [ faldo:begin [ faldo:position ?begin ; faldo:reference ?chr ] ;
                          faldo:end   [ faldo:position ?end ] ] .
    OPTIONAL {
      ?cre fantao:hasTSS [ a fantao:NearestTSS ;
                           sio:SIO_000216 [ a fantao:TSSDistance ; sio:SIO_000300 ?tss_distance ] ] .
    }
  }
}`;

  const tf_query = (ncbigene) => `${PREFIXES}
SELECT DISTINCT ?cre_id ?symbol ?maxq
WHERE {
  GRAPH ${FANTA_GRAPH} {
    ?reg rdfs:seeAlso <http://identifiers.org/ncbigene/${ncbigene}> .
    ?cre sio:SIO_000395 ?reg ;
         dct:identifier ?cre_id ;
         fantao:hasOverlappedAnnotation ?annotation .
    ?annotation a fantao:ChipAtlasAntigen ;
                rdfs:label ?symbol ;
                sio:SIO_000216 [ a fantao:MaxQscore ; sio:SIO_000300 ?maxq ] .
  }
}`;

  // hco:<chromosome>/<assembly>
  const chrom_of = (iri) => {
    const m = /\/([^/]+)\/[^/]+$/.exec(iri || "");
    return m ? "chr" + m[1] : null;
  };

  const fanta_cres = async (ncbigene) => {
    const [rows, tf_rows] = await Promise.all([ask(cre_query(ncbigene)), ask(tf_query(ncbigene))]);
    const by_id = new Map();
    for (const b of rows) {
      const id = val(b, "cre_id");
      if (by_id.has(id)) continue;
      by_id.set(id, {
        cre_id: id,
        cre_name: val(b, "cre_name"),
        cre_chrom: chrom_of(val(b, "chr")),
        cre_start: Number(val(b, "begin")),
        cre_end: Number(val(b, "end")),
        cre_tss_distance: val(b, "tss_distance"),
        tf: []
      });
    }
    // Highest q-score first, as the JSON Lines file listed them.
    const seen = new Set();
    for (const b of tf_rows.sort((x, y) => Number(val(y, "maxq")) - Number(val(x, "maxq")))) {
      const id = val(b, "cre_id");
      const key = id + "\t" + val(b, "symbol");
      if (!by_id.has(id) || seen.has(key)) continue;
      seen.add(key);
      by_id.get(id).tf.push({symbol: val(b, "symbol"), maxq: val(b, "maxq")});
    }
    return [...by_id.values()];
  };

  const [human_cre, mouse_cre] = await Promise.all([
    fanta_cres(pair.human.ncbigene),
    fanta_cres(pair.mouse.ncbigene)
  ]);

  //// ---- coordinates ------------------------------------------------------
  // Four batches: each species' CREs to the other species, and each species'
  // CREs to GRCm39, which is the assembly MoG+ 2.1 and 3 are built on. The
  // mouse GRCm39 position is taken straight from GRCm38 rather than through
  // the human genome, so it keeps the whole CRE instead of only the part that
  // happens to align to human.
  const [h_to_m, m_to_h, h_to_m39, m_to_m39] = await Promise.all([
    convert(human_cre.map(d => d.cre_id), 10090, "mm10"),
    convert(mouse_cre.map(d => d.cre_id), 9606, "hg38"),
    convert(human_cre.map(d => d.cre_id), 10090, "GRCm39"),
    convert(mouse_cre.map(d => d.cre_id), 10090, "GRCm39")
  ]);

  const apply = (list, lift, to_taxon, from_taxon) => {
    list.forEach((cre, i) => {
      const item = lift[i];
      if (!item || item.error) return;
      const input = parse_location(item.input);
      if (!input) return;
      cre.cre_chrom = chrom_name(input.sequence, from_taxon) || cre.cre_chrom;
      cre.cre_start = input.low;
      cre.cre_end = input.high;
      cre.cre_sequence = input.sequence;
      cre.cre_assembly = item.inputAssembly;
      const hit = (item.results || []).find(r => r.category == "genome") || (item.results || [])[0];
      if (!hit) return;
      const lifted = parse_location(hit.location);
      const [ms, me] = matched_range((hit.path || [])[0], input.low, input.high);
      cre.match_start = ms;
      cre.match_end = me;
      cre.lift_chr = chrom_name(lifted.sequence, to_taxon);
      // Written along the strand, so start > end on the complementary strand.
      cre.lift_start = lifted.complement ? lifted.high : lifted.low;
      cre.lift_end = lifted.complement ? lifted.low : lifted.high;
      cre.lift_assembly = hit.assembly;
      cre.lift_aligned = lifted.aligned;
      cre.lift_blocks = lifted.blocks.length;
      cre.lift_approximate = !!hit.approximate;
    });
  };

  const apply_m39 = (list, lift) => {
    list.forEach((cre, i) => {
      const item = lift[i];
      if (!item || item.error) return;
      const hit = (item.results || []).find(r => r.category == "genome") || (item.results || [])[0];
      if (!hit) return;
      const m39 = parse_location(hit.location);
      cre.mm39_chrom = chrom_name(m39.sequence, 10090);
      cre.mm39_start = m39.low;
      cre.mm39_end = m39.high;
      cre.mm39_aligned = m39.aligned;
      cre.mm39_blocks = m39.blocks.length;
    });
  };

  apply(human_cre, h_to_m, 10090, 9606);
  apply(mouse_cre, m_to_h, 9606, 10090);
  apply_m39(human_cre, h_to_m39);
  apply_m39(mouse_cre, m_to_m39);

  //// ---- the genes themselves ---------------------------------------------
  const window_of = (own, other) => {
    const chrom = (own.find(d => d.cre_chrom) || {}).cre_chrom;
    // Only positions on this track's own chromosome. A lifted position can land
    // on another one -- the hg38-to-mm39 chain puts part of BRCA1 on mouse
    // chr8, a duplicated region -- and mixing that in stretched the mouse
    // window from 27 kb to 5 Mb, far enough that the annotation call came back
    // truncated with the gene itself cut off.
    const p = own.filter(d => d.cre_chrom == chrom).flatMap(d => [d.cre_start, d.cre_end])
      .concat(other.filter(d => d.lift_chr == chrom).flatMap(d => [d.lift_start, d.lift_end]))
      .filter(x => x != null).map(Number);
    if (!p.length) return null;
    return [p.reduce((a, b) => Math.min(a, b)), p.reduce((a, b) => Math.max(a, b))];
  };
  const sequence_of = (list) => (list.find(d => d.cre_sequence) || {}).cre_sequence;
  const h_win = window_of(human_cre, mouse_cre);
  const m_win = window_of(mouse_cre, human_cre);
  const [human_gene, mouse_gene] = await Promise.all([
    h_win ? gene_feature(sequence_of(human_cre), h_win[0], h_win[1], pair.human.ncbigene) : null,
    m_win ? gene_feature(sequence_of(mouse_cre), m_win[0], m_win[1], pair.mouse.ncbigene) : null
  ]);

  //// ---- TF list for the ortholog query ------------------------------------
  const human_tf = [...new Set(human_cre.flatMap(d => d.tf.map(a => a.symbol)))];
  const tgid = "https://api.togoid.dbcls.jp/convert?route=hgnc_symbol%2Chgnc%2Cncbigene&report=target&format=json&ids="
    + human_tf.join("%2C");
  const tf_ids = human_tf.length ? (await fetch(tgid).then(r => r.json())).results : [];

  return {
    human: human_cre.sort((a, b) => a.cre_start - b.cre_start),
    mouse: mouse_cre.sort((a, b) => a.cre_start - b.cre_start),
    human_tf: tf_ids.length ? "ncbigene:" + tf_ids.join(" ncbigene:") : "ncbigene:0",
    orth: pair,
    entry: {species: entry_species, type: gene.entry.type, id: gene.entry.id},
    gene: {human: human_gene, mouse: mouse_gene}
  };
}
```

## `tf_orth`
```sparql
PREFIX orth: <http://purl.org/net/orth#>
PREFIX taxid: <http://identifiers.org/taxonomy/>
PREFIX ncbigene: <http://identifiers.org/ncbigene/>
PREFIX : <https://dbcls.github.io/ncbigene-rdf/ontology.ttl#>

SELECT DISTINCT ?human ?human_label ?mouse ?mouse_label
WHERE {
  VALUES ?human { {{cre.human_tf}} }
  ?human orth:hasOrtholog+ ?mouse ;
         :taxid taxid:9606 ;
         rdfs:label ?human_label .
  ?mouse :taxid taxid:10090 ;
         rdfs:label ?mouse_label .
}
```

## `return`
```javascript
({cre, tf_orth}) => {
  let human2mouse = {};
  let mouse2human = {};
  tf_orth.results.bindings.forEach(d => {
    human2mouse[d.human_label.value] = d.mouse_label.value;
    mouse2human[d.mouse_label.value] = d.human_label.value;
  });

  let cre_comparison = [];
  for (const [m_i, m_cre] of cre.mouse.entries()) {
    // Built fresh for every pair. The previous version built this once per
    // mouse CRE and deleted from it as human CREs matched, so each shared TF
    // was credited to the first human CRE only and every later pair lost it.
    const m_symbols = new Set(m_cre.tf.map(t => t.symbol));
    const mouse_tf_has_orth = [...m_symbols].filter(s => mouse2human[s]).length;

    for (const [h_i, h_cre] of cre.human.entries()) {
      const h_symbols = new Set(h_cre.tf.map(t => t.symbol));
      const human_tf_has_orth = [...h_symbols].filter(s => human2mouse[s]).length;
      const and_tf = [...h_symbols].filter(s => human2mouse[s] && m_symbols.has(human2mouse[s]));
      const and = and_tf.length;
      if (and == 0) continue;
      const or = mouse_tf_has_orth + human_tf_has_orth - and;
      cre_comparison.push({
        human_cre: h_cre.cre_id,
        human_cre_index: h_i,
        mouse_cre: m_cre.cre_id,
        mouse_cre_index: m_i,
        human_tf_count: h_cre.tf.length,
        mouse_tf_count: m_cre.tf.length,
        human_tf_has_orth: human_tf_has_orth,
        mouse_tf_has_orth: mouse_tf_has_orth,
        and: and,
        or: or,
        jaccard: or > 0 ? Math.round(1000 * and / or) / 1000 : 0,
        and_tf: and_tf
      });
    }
  }

  return {
    orth: cre.orth,
    entry: cre.entry,
    gene: cre.gene,
    cre_comparison: cre_comparison,
    human_cre: cre.human,
    mouse_cre: cre.mouse
  };
}
```
