# CRE variants: TogoVar (GRCh38) joined with MoG+ (GRCm39)

Successor to `togovar_cre_mogplus`, which is left in place untouched.

* What changed
  * The GRCh38 <-> GRCm39 conversion is done by **TogoCoord** instead of
    `orth.dbcls.jp/cgi-bin/liftOver`, and the mouse strand comes from the
    conversion rather than from parsing the first line of a chain dump. The old
    parser read only lines beginning `hg`, so a region that maps to several
    chains -- the CGI answers `MULTIPLE_HITS` -- produced no coordinates at all.
  * Accepts a **mouse GRCm39 range** as well as a human GRCh38 one. A mouse CRE
    therefore no longer has to be routed `GRCm38 -> GRCh38 -> GRCm39`: the
    detour through human silently dropped every CRE with no human counterpart
    and truncated the rest to the part that aligns to human.
  * **Takes several ranges at once** and merges the ones within `merge_gap` of
    each other. A gene page used to call this once per CRE -- 43 calls for
    CTNNB1, each re-fetching the strain list and hitting MoG+ at RIKEN
    separately, around 258 external requests for one page. Merged, CTNNB1's 43
    ranges become 4 windows and 13 requests; across 15 genes the range of
    requests is 7 to 19. MoG+ answers a 100 kb window in about the same time as
    a 400 bp one, so nothing is lost by asking for more at once.
    Checked against the single-range version on those 15 genes: same rows, and
    the same per-CRE variant, match and strain counts.
  * Rows are returned even when no human variant pairs with the mouse one, so
    mouse-only variation in a CRE is visible; `alt_match` is then `-`.
  * Removed the duplicate `chrName=5` left in both MoG+ URLs. MoG+ honours the
    first occurrence, so it was inert, but it was pinned to the chromosome of
    the gene the query was written against.

## Returns

One row per (human allele x mouse variant) pair found, plus a row per mouse
variant that pairs with nothing. Each row carries `in_range`: which of the
requested ranges it belongs to, so a caller that asked about several CREs can
hand the rows back out. A row outside every requested range -- merging two
ranges into one window also picks up what lies between them -- has an empty
`in_range`.

The same mouse variant can appear both paired and unpaired, for different
ranges. Its human counterpart is wherever it lifts to, which for a mouse range
is the answer wanted; but a human range that reaches the variant only through
its own mouse footprint must not be credited with a human variant sitting
somewhere else in the genome, and gets the unpaired row instead.

## Parameters

* `ranges` : GRCh38 positions, comma separated
  * default: chr12:111767166-111767813
  * example: chr12:111766858-111767071,chr12:111792130-111792764
* `mmu_ranges` : GRCm39 positions, comma separated. Both may be given.
  * default:
  * example: chr5:121731438-121731993
* `merge_gap` : ranges closer than this are queried as one window
  * default: 1000
* `max_window` : a single window is never widened past this
  * default: 200000
* `mogplus_ver` : mogplus21 or mogplus3
  * default: mogplus21
* `togocoord`
  * default: https://togocoord.dbcls.jp/

## `sequence`
```javascript
async ({ranges, mmu_ranges, range, mmu_range, merge_gap, max_window, mogplus_ver, togocoord}) => {
  const base = (togocoord || "https://togocoord.dbcls.jp/").replace(/\/*$/, "/");
  const gap = Number(merge_gap ?? 1000);
  // A ceiling on how wide one MoG+ query may get. Real CRE windows come to a
  // few kb, so this never binds; it is there so that a chain of merges, or an
  // alignment whose blocks straddle a megabase, cannot turn into one enormous
  // request against MoG+.
  const max_span = Number(max_window ?? 200000);
  // `range` and `mmu_range` are the single-range spelling this API started with.
  const human_in = String(ranges || range || "").split(",").map(x => x.trim()).filter(Boolean);
  const mouse_in = String(mmu_ranges || mmu_range || "").split(",").map(x => x.trim()).filter(Boolean);
  if (!human_in.length && !mouse_in.length) return "No range given";

  const sort_consequences = (s) => {
    const order = ["transcript_ablation","splice_acceptor_variant","splice_donor_variant","stop_gained",
      "frameshift_variant","stop_lost","start_lost","transcript_amplification","feature_elongation",
      "feature_truncation","inframe_insertion","inframe_deletion","missense_variant","protein_altering_variant",
      "splice_donor_5th_base_variant","splice_region_variant","splice_donor_region_variant",
      "splice_polypyrimidine_tract_variant","incomplete_terminal_codon_variant","start_retained_variant",
      "stop_retained_variant","synonymous_variant","coding_sequence_variant","mature_miRNA_variant",
      "5_prime_UTR_variant","3_prime_UTR_variant","non_coding_transcript_exon_variant","intron_variant",
      "NMD_transcript_variant","non_coding_transcript_variant","coding_transcript_variant","upstream_gene_variant",
      "downstream_gene_variant","TFBS_ablation","TFBS_amplification","TF_binding_site_variant",
      "regulatory_region_ablation","regulatory_region_amplification","regulatory_region_variant",
      "intergenic_variant","sequence_variant"];
    return (s || "").split(",").map(x => x.trim())
      .sort((a, b) => order.indexOf(a) - order.indexOf(b)).join("<br/>");
  };

  // Only real bases are complemented; the "-" of an indel and anything
  // unexpected pass through, where the old conv_nt returned undefined.
  const conv_nt = (strand, nt) => {
    if (strand != "-") return nt;
    const c = {A: "T", T: "A", C: "G", G: "C"};
    return String(nt).split("").reverse().map(x => c[x] || x).join("");
  };

  const parse_location = (loc) => {
    if (!loc) return null;
    const first = loc.indexOf(":"), second = loc.indexOf(":", first + 1);
    if (second < 0) return null;
    const body = loc.slice(second + 1);
    const blocks = [];
    for (const part of body.replace(/[a-z]+\(/g, "").replace(/[)<>]/g, "").split(",")) {
      const m = part.match(/^\s*(\d+)(?:(?:\.\.|\^|\.)(\d+))?\s*$/);
      if (!m) continue;
      const a = parseInt(m[1], 10), c = m[2] === undefined ? a : parseInt(m[2], 10);
      blocks.push({start: Math.min(a, c), end: Math.max(a, c)});
    }
    if (!blocks.length) return null;
    // reduce rather than Math.min(...blocks): V8 caps a spread at about
    // 124,000 arguments, and a long join would be close enough to matter.
    const low = blocks.reduce((m, x) => Math.min(m, x.start), Infinity);
    const high = blocks.reduce((m, x) => Math.max(m, x.end), -Infinity);
    return {sequence: loc.slice(0, second), complement: body.includes("complement("), blocks, low, high};
  };

  const chrom_name = (sequence, taxon) => {
    const m = /NC_(\d+)\.\d+$/.exec(sequence || "");
    if (!m) return null;
    const n = parseInt(m[1], 10);
    if (taxon == 9606) return n <= 22 ? String(n) : ({23: "X", 24: "Y"}[n] || String(n));
    if (taxon == 10090) return n >= 67 && n <= 85 ? String(n - 66) : ({86: "X", 87: "Y"}[n] || String(n));
    return null;
  };

  const parse_range = (s) => {
    const m = /^(?:chr)?([^:]+):(\d+)-(\d+)$/.exec(s);
    if (!m) return null;
    const a = Number(m[2]), b = Number(m[3]);
    return {chr: m[1], start: Math.min(a, b), end: Math.max(a, b), text: s};
  };

  const convert = async (locations, taxon, assembly) => {
    if (!locations.length) return [];
    const out = [];
    // /v1/convert takes at most 1000 locations, so it is chunked.
    for (let i = 0; i < locations.length; i += 1000) {
      const res = await fetch(base + "v1/convert", {
        method: "POST", headers: {"Content-Type": "application/json"},
        body: JSON.stringify({locations: locations.slice(i, i + 1000), to: "genome",
                              taxon: String(taxon), assembly})
      });
      if (!res.ok) throw new Error("TogoCoord " + res.status);
      out.push(...((await res.json()).results || []));
    }
    return out;
  };
  const as_loc = (assembly, chr, from, to) =>
    assembly + ":chr" + String(chr).replace(/^chr/, "") + ":" + from + ".." + to;
  // The first genome hit. When a region matches several chains TogoCoord can
  // return more than one, and the first is not necessarily the best: part of
  // BRCA1 comes back as mouse chr8, a duplicated region, ahead of chr11. Once
  // TogoCoord reports how much of the input each hit actually covers, pick the
  // largest here instead.
  const genome_hit = (r) => (r && !r.error ? (r.results || []).find(x => x.category == "genome") : null);

  //// ---- the strain list, once for the whole call ------------------------
  // A page used to re-fetch these two per CRE. They do not depend on the
  // ranges, so they are started here and awaited after the conversions.
  const mogplus = mogplus_ver || "mogplus21";
  const options = {method: "GET", headers: {Accept: "application/json"}};
  const strains = Promise.all([
    fetch("https://grch38.togovar.org/sparqlist/api/mouse_strain?strain_id=&strain=all", {
      headers: {Accept: "application/json", "Content-Type": "application/json"}}).then(r => r.json()),
    fetch("https://grch38.togovar.org/sparqlist/api/mouse_strain?strain_id=all", options).then(r => r.json())
  ]);

  //// ---- one set of windows, in mouse coordinates -------------------------
  // MoG+ is the only source keyed on the mouse genome, so the windows are
  // merged there. Merging each species separately would let a human window and
  // a mouse window cover the same stretch and report its variants twice.
  const human_ranges = human_in.map(parse_range).filter(Boolean);
  const mouse_ranges = mouse_in.map(parse_range).filter(Boolean);

  const to_mouse = await convert(
    human_ranges.map(r => as_loc("GRCh38", r.chr, r.start, r.end)), 10090, "GRCm39");
  // Ranges within `gap` become one interval, and none grows past `max_span`.
  const merge = (list) => {
    const out = [];
    for (const r of list.slice().sort((a, b) => a.start - b.start)) {
      const last = out[out.length - 1];
      if (last && r.start - last.end <= gap && r.end - last.start + 1 <= max_span)
        last.end = Math.max(last.end, r.end);
      else out.push({start: r.start, end: r.end});
    }
    return out;
  };

  const on_mouse = [];
  human_ranges.forEach((r, i) => {
    const hit = genome_hit(to_mouse[i]);
    if (!hit) return;                       // no mouse counterpart: nothing to ask MoG+
    const p = parse_location(hit.location);
    const chr = chrom_name(p.sequence, 10090);
    if (!chr) return;
    // Normally the whole span, which is what the single-range API used. Only
    // when the alignment is scattered far too widely to query as one piece does
    // it fall back to the blocks the conversion actually aligned.
    const pieces = p.high - p.low + 1 <= max_span
      ? [{start: p.low, end: p.high}] : merge(p.blocks);
    for (const piece of pieces) on_mouse.push({chr, start: piece.start, end: piece.end, human: r});
  });
  for (const r of mouse_ranges) on_mouse.push({chr: r.chr, start: r.start, end: r.end, human: null});

  const by_chr = {};
  for (const r of on_mouse) (by_chr[r.chr] ??= []).push(r);
  const windows = [];
  for (const chr of Object.keys(by_chr)) {
    let cur = null;
    for (const r of by_chr[chr].sort((a, b) => a.start - b.start)) {
      if (cur && r.start - cur.mmu.end <= gap && r.end - cur.mmu.start + 1 <= max_span) {
        cur.mmu.end = Math.max(cur.mmu.end, r.end);
        cur.parts.push(r);
      } else {
        cur = {mmu: {chr, start: r.start, end: r.end}, parts: [r]};
        windows.push(cur);
      }
    }
  }
  if (!windows.length) return "Liftover position not found";

  // The human side of each window, which is what TogoVar is asked for.
  //
  // Where human ranges fed the window, those ranges are the human side: the
  // round trip is not trustworthy enough to widen them with. The hg38-to-mm39
  // chain puts part of BRCA1 on mouse chr8, a duplicated region, and converting
  // that back lands on human chr2 -- so a window built from a chr17 CRE would
  // otherwise be paired against chr2 variants. The round trip is used only for
  // a window with no human range of its own, where it is the only thing there
  // is, and then only to the extent it agrees on the chromosome.
  const back = await convert(
    windows.map(w => as_loc("GRCm39", w.mmu.chr, w.mmu.start, w.mmu.end)), 9606, "GRCh38");
  windows.forEach((w, i) => {
    const hit = genome_hit(back[i]);
    let round_trip = null;
    if (hit) {
      const p = parse_location(hit.location);
      const chr = chrom_name(p.sequence, 9606);
      if (chr) round_trip = {chr, start: p.low, end: p.high};
      w.strand = hit.orientation == "reverse" ? "-" : "+";
    }
    if (!w.strand) w.strand = "+";

    const own = w.parts.filter(x => x.human).map(x => x.human);
    let spans;
    if (own.length) {
      // Whichever chromosome most of them are on; they merged in mouse space,
      // which does not promise they share one in human.
      const count = {};
      for (const r of own) count[r.chr] = (count[r.chr] || 0) + 1;
      const chr = Object.keys(count).sort((a, b) => count[b] - count[a])[0];
      spans = own.filter(r => r.chr == chr);
      if (round_trip && round_trip.chr == chr) spans.push(round_trip);
    } else {
      spans = round_trip ? [round_trip] : [];
    }
    // Merged the same way as the mouse windows, so neighbouring CREs share one
    // query while distant ones keep their own rather than a span across the gap.
    const chr_of = spans.length ? spans[0].chr : null;
    w.hsa = merge(spans).map(x => ({chr: chr_of, start: x.start, end: x.end}));
  });

  //// ---- MoG+ and TogoVar, once per window ---------------------------------
  const [strain2id, mmu_strains] = await strains;
  const strain_ids = [];
  for (const d of Object.keys(mmu_strains)) {
    if ((mmu_strains[d].category == "mogplus3" && mogplus == "mogplus21")
        || (mmu_strains[d].category == "mogplus21" && mogplus == "mogplus3")) continue;
    strain_ids.push(encodeURIComponent(mmu_strains[d].id));
  }
  const strain_param = "strainNoSlct=" + strain_ids.join("&strainNoSlct=");

  await Promise.all(windows.map(async (w) => {
    // The two are independent, and a window with no MoG+ variant must still
    // fetch its human variants.
    const [mog, tgv] = await Promise.all([
      fetch("https://molossinus.brc.riken.jp/" + mogplus + "/variantTable/?" + strain_param
        + "&chrName=" + w.mmu.chr + "&chrStart=" + w.mmu.start + "&chrEnd=" + w.mmu.end
        + "&seqType=genome&geneNameSearchText=&index=submit&presentType=dwnld").then(d => d.text()),
      Promise.all(w.hsa.map(h => fetch("https://sparql-support.dbcls.jp/api/getTogoVarGeneAnn?q="
        + h.chr + ":" + h.start + "-" + h.end).then(d => d.text())))
    ]);
    w.togovar = tgv.join("\n");
    const rows = mog.split(/\n/);
    const pos_list = rows[0].split(/\t/);
    w.variants = {};
    if (!pos_list[1]) return;
    const ref_list = rows[1] ? rows[1].split(/\t/) : [];
    const col2pos = [];
    for (let i = 1; i < pos_list.length; i++) {
      col2pos[i] = pos_list[i];
      let s = Math.max(1, i - 20), e = Math.min(pos_list.length - 1, i + 20);
      let from = Number(pos_list[s]), to = Number(pos_list[e]);
      if (to < from) { const t = to; to = from; from = t; }
      w.variants[pos_list[i]] = {ref: ref_list[i], alt: [], strain: {},
                                 region: "&chrStart=" + from + "&chrEnd=" + to};
    }
    for (let i = 2; i < rows.length; i++) {
      const cells = rows[i].split(/\t/);
      for (let j = 1; j < cells.length; j++) {
        if (!cells[j].match(/[ATCG-]/)) continue;
        const pos = col2pos[j];
        if (!w.variants[pos].alt.includes(cells[j])) w.variants[pos].alt.push(cells[j]);
        (w.variants[pos].strain[cells[j]] ??= []).push(cells[0]);
      }
    }
  }));

  //// ---- every MoG+ position back to GRCh38, pooled ------------------------
  // One request for the whole page rather than one per CRE. The positions have
  // to go in one at a time: a join of them comes back with adjacent blocks
  // coalesced, which loses the correspondence to the input.
  const wanted = [];
  for (const w of windows) for (const p of Object.keys(w.variants || {})) wanted.push([w.mmu.chr, p]);
  const uniq = [...new Set(wanted.map(([c, p]) => c + ":" + p))];
  const mapped = await convert(uniq.map(k => {
    const [c, p] = k.split(":");
    return as_loc("GRCm39", c, p, p);
  }), 9606, "GRCh38");
  const mmu2hsa = {};
  uniq.forEach((k, i) => {
    const hit = genome_hit(mapped[i]);
    if (!hit) return;
    const p = parse_location(hit.location);
    if (p) mmu2hsa[k] = {chr: chrom_name(p.sequence, 9606), pos: p.low,
                         strand: hit.orientation == "reverse" ? "-" : "+"};
  });

  //// ---- join ---------------------------------------------------------------
  const strain_cell = (names) => {
    const labels = [], ids = [];
    for (const n of names || []) {
      ids.push(strain2id[n] ? strain2id[n].id : n);
      labels.push(strain2id[n] && strain2id[n].source
        ? "<a href=" + strain2id[n].source + ">" + n + "</a>" : n);
    }
    return {labels: labels.join("<br/>"), ids};
  };
  const mog_url = (w, pos, ids) =>
    "https://molossinus.brc.riken.jp/" + mogplus + "/variantTable/?strainNoSlct=refGenome&strainNoSlct="
    + ids.join("&strainNoSlct=").replace(/\//g, "_") + "&chrName=" + w.mmu.chr + w.variants[pos].region
    + "&seqType=genome&geneNameSearchText=&index=submit&presentType=disp";

  // Which of the ranges the caller asked for does this variant fall in? Asked
  // globally rather than per window, because merging mixed the two species'
  // ranges into shared windows.
  //
  // `paired` are the ranges the human allele may be reported against: a mouse
  // range, whose human counterpart is wherever it lifts to, and a human range
  // that contains the human position. `mouse_only` are human ranges reached the
  // other way round -- the variant sits in the range's mouse footprint, so it
  // belongs to that CRE, but its human counterpart is somewhere else in the
  // genome and must not be counted as a variant of that CRE. The single-range
  // API drew the same line by accident: asked about one human range it queried
  // TogoVar over exactly that range, so a mouse variant lifting to elsewhere
  // came back unpaired.
  const covers = (w, mmu_chr, mmu_pos, hsa_chr, hsa_pos) => {
    const paired = new Set(), mouse_only = new Set();
    for (const r of mouse_ranges)
      if (r.chr == mmu_chr && r.start <= mmu_pos && mmu_pos <= r.end) paired.add(r.text);
    if (hsa_pos != null) for (const r of human_ranges)
      if (r.chr == hsa_chr && r.start <= hsa_pos && hsa_pos <= r.end) paired.add(r.text);
    for (const part of w.parts)
      if (part.human && part.start <= mmu_pos && mmu_pos <= part.end
          && !paired.has(part.human.text)) mouse_only.add(part.human.text);
    return {paired: [...paired], mouse_only: [...mouse_only]};
  };

  const rows = [];
  for (const w of windows) {
    const hsa2mmu = {};
    for (const p of Object.keys(w.variants || {})) {
      const m = mmu2hsa[w.mmu.chr + ":" + p];
      if (m) hsa2mmu[m.pos] = p;
    }

    // The mouse-only shape of a variant: what there is to say about it without
    // a human allele.
    const mouse_row = (pos, mmu_alt, in_range) => {
      const s = strain_cell(w.variants[pos].strain[mmu_alt]);
      const h = mmu2hsa[w.mmu.chr + ":" + pos];
      return {
        tgv_id: "", tgv_link: "", rs: "", rs_link: "",
        allele_grch38: h && h.chr ? h.chr + ":" + h.pos + "-?-?" : "",
        consequence: "",
        mmu_strand: h ? h.strand : w.strand,
        allele_grcm39: w.mmu.chr + ":" + pos + "-" + w.variants[pos].ref + "-" + mmu_alt,
        ref_grcm39: w.variants[pos].ref,
        alt_grcm39: mmu_alt,
        alt_match: "-",
        mouse_strains: s.labels,
        mogplus_url: mog_url(w, pos, s.ids),
        in_range
      };
    };

    const paired = {};       // mouse alleles that found a human variant
    const unpaired = {};     // ...and the ranges that must still see them bare
    (w.togovar || "").split("\n").forEach(line => {
      if (!line) return;
      const [tgv_id, rs, chr, hsa_pos, hsa_ref, hsa_alt, symbol, transcript_id, consequence] = line.split(/\t/);
      const mmu_pos = hsa2mmu[hsa_pos];
      if (!mmu_pos || !w.variants[mmu_pos]) return;
      const strand = (mmu2hsa[w.mmu.chr + ":" + mmu_pos] || {}).strand || w.strand;
      const mmu_ref_orig = w.variants[mmu_pos].ref;
      if (conv_nt(strand, mmu_ref_orig) != hsa_ref || !w.variants[mmu_pos].alt[0]) return;
      if (hsa_ref.length != 1 || hsa_alt.length != 1) return;
      const where = covers(w, w.mmu.chr, Number(mmu_pos), chr, Number(hsa_pos));
      for (const mmu_alt of w.variants[mmu_pos].alt) {
        const k = mmu_pos + "\t" + mmu_alt;
        paired[k] = true;
        for (const text of where.mouse_only) (unpaired[k] ??= new Set()).add(text);
        const s = strain_cell(w.variants[mmu_pos].strain[mmu_alt]);
        rows.push({
          tgv_id: tgv_id == "-" ? "" : tgv_id,
          tgv_link: tgv_id == "-" ? "" : "/variant/" + tgv_id,
          rs: rs == "-" ? "" : rs,
          rs_link: rs == "-" ? "" : "https://www.ncbi.nlm.nih.gov/snp/" + rs,
          allele_grch38: chr + ":" + hsa_pos + "-" + hsa_ref + "-" + hsa_alt,
          consequence: sort_consequences(consequence),
          mmu_strand: strand,
          allele_grcm39: w.mmu.chr + ":" + mmu_pos + "-" + mmu_ref_orig + "-" + mmu_alt,
          ref_grcm39: mmu_ref_orig,
          alt_grcm39: mmu_alt,
          alt_match: hsa_alt == conv_nt(strand, mmu_alt) ? "Yes" : "No",
          mouse_strains: s.labels,
          mogplus_url: mog_url(w, mmu_pos, s.ids),
          in_range: where.paired
        });
      }
    });

    for (const pos of Object.keys(w.variants || {})) {
      for (const mmu_alt of w.variants[pos].alt) {
        const k = pos + "\t" + mmu_alt;
        if (paired[k]) {
          // Paired, but some CRE reaches it only through its mouse footprint.
          if (unpaired[k]) rows.push(mouse_row(pos, mmu_alt, [...unpaired[k]]));
          continue;
        }
        const h = mmu2hsa[w.mmu.chr + ":" + pos];
        const where = covers(w, w.mmu.chr, Number(pos), h ? h.chr : null, h ? h.pos : null);
        rows.push(mouse_row(pos, mmu_alt, [...where.paired, ...where.mouse_only]));
      }
    }
  }

  if (!rows.length) return "MoG+ variant not found";
  // Pairs first, then mouse-only, each by position.
  rows.sort((a, b) => (a.alt_match == "-") - (b.alt_match == "-")
    || Number(a.allele_grcm39.split(/[:-]/)[1]) - Number(b.allele_grcm39.split(/[:-]/)[1]));
  return rows;
}
```
