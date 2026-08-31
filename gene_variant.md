# Gene report / Variant

## Parameters

* `hgnc_id` HGNC ID
  * default: 404
* `offset` number of skip records
  * default: 0

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_APP, hgnc_id, offset}) => {
  const RECORDS_PER_PAGE = 100;
  const OFFSET_MAX = 10000;

  const parsedOffset = Number.parseInt(offset, 10);
  offset = Number.isFinite(parsedOffset) ? Math.max(0, parsedOffset) : 0;

  if (offset + RECORDS_PER_PAGE > OFFSET_MAX) {
    return [];
  }

  if (!String(hgnc_id).match(/^\d+$/)) {
    return [];
  }

  const query = {
    query: {
      gene: {
        relation: "eq",
        terms: [Number.parseInt(hgnc_id, 10)],
      },
    },
    offset: offset,
    limit: RECORDS_PER_PAGE,
  };

  let data = [];

  try {
    const res = await fetch(
      SPARQLIST_TOGOVAR_APP.concat("/api/search/variant"),
      {
        method: "POST",
        headers: {
          Accept: "application/json",
          "Content-Type": "application/json",
        },
        body: JSON.stringify(query),
      },
    );

    if (!res.ok) {
      console.error("Failed to fetch variants", {
        status: res.status,
        statusText: res.statusText,
      });
      return [];
    }

    const json = await res.json();
    data = Array.isArray(json)
      ? json
      : json && Array.isArray(json.data)
        ? json.data
        : [];
  } catch (error) {
    console.error("Failed to fetch variants", error);
    return [];
  }

  const typeLabel = {
    SO_0001483: "SNV",
    SO_0000667: "Insertion",
    SO_0000159: "Deletion",
    SO_1000032: "Indel",
    SO_0002007: "MNV",
  };

  const consequenceLabel = {
    SO_0001893: { label: "Transcript ablation", order: 1 },
    SO_0001574: { label: "Splice acceptor variant", order: 2 },
    SO_0001575: { label: "Splice donor variant", order: 3 },
    SO_0001587: { label: "Stop gained", order: 4 },
    SO_0001589: { label: "Frameshift variant", order: 5 },
    SO_0001578: { label: "Stop lost", order: 6 },
    SO_0002012: { label: "Start lost", order: 7 },
    SO_0001889: { label: "Transcript amplification", order: 8 },
    SO_0001821: { label: "Inframe insertion", order: 9 },
    SO_0001822: { label: "Inframe deletion", order: 10 },
    SO_0001583: { label: "Missense variant", order: 11 },
    SO_0001818: { label: "Protein altering variant", order: 12 },
    SO_0001630: { label: "Splice region variant", order: 13 },
    SO_0001626: { label: "Incomplete terminal codon variant", order: 14 },
    SO_0002019: { label: "Start retained variant", order: 15 },
    SO_0001567: { label: "Stop retained variant", order: 16 },
    SO_0001819: { label: "Synonymous variant", order: 17 },
    SO_0001580: { label: "Coding sequence variant", order: 18 },
    SO_0001620: { label: "Mature miRNA variant", order: 19 },
    SO_0001623: { label: "5 prime UTR variant", order: 20 },
    SO_0001624: { label: "3 prime UTR variant", order: 21 },
    SO_0001792: { label: "Non coding transcript exon variant", order: 22 },
    SO_0001627: { label: "Intron variant", order: 23 },
    SO_0001621: { label: "NMD transcript variant", order: 24 },
    SO_0001619: { label: "Non coding transcript variant", order: 25 },
    SO_0001631: { label: "Upstream gene variant", order: 26 },
    SO_0001632: { label: "Downstream gene variant", order: 27 },
    SO_0001895: { label: "TFBS ablation", order: 28 },
    SO_0001892: { label: "TFBS amplification", order: 29 },
    SO_0001782: { label: "TF binding site variant", order: 30 },
    SO_0001894: { label: "Regulatory region ablation", order: 31 },
    SO_0001891: { label: "Regulatory region amplification", order: 32 },
    SO_0001907: { label: "Feature elongation", order: 33 },
    SO_0001566: { label: "Regulatory region variant", order: 34 },
    SO_0001906: { label: "Feature truncation", order: 35 },
    SO_0001628: { label: "Intergenic variant", order: 36 },
  };

  const clinicalSignificanceLabel = {
    P: "Pathogenic",
    LP: "Likely pathogenic",
    PLP: "Pathogenic, Low penetrance",
    LPLP: "Likely pathogenic, Low penetrance",
    ERA: "Established risk allele",
    LRA: "Likely risk allele",
    URA: "Uncertain risk allele",
    US: "Uncertain significance",
    LB: "Likely benign",
    B: "Benign",
    CC: "Conflicting classifications of pathogenicity",
    DR: "Drug response",
    CS: "Confers sensitivity",
    RF: "Risk factor",
    A: "Association",
    PR: "Protective",
    AF: "Affects",
    O: "Other",
    NP: "Not provided",
    AN: "Association not found",
  };

  const clinicalSignificanceKey = {
    pathogenic: "P",
    "likely pathogenic": "LP",
    "pathogenic, low penetrance": "PLP",
    "likely pathogenic, low penetrance": "LPLP",
    "established risk allele": "ERA",
    "likely risk allele": "LRA",
    "uncertain risk allele": "URA",
    "uncertain significance": "US",
    "likely benign": "LB",
    benign: "B",
    "conflicting interpretations of pathogenicity": "CC",
    "conflicting classifications of pathogenicity": "CC",
    "drug response": "DR",
    "confers sensitivity": "CS",
    "risk factor": "RF",
    association: "A",
    protective: "PR",
    affects: "AF",
    other: "O",
    "not provided": "NP",
    "no assertion provided": "NP",
    "no classification provided": "NP",
    "association not found": "AN",
    association_not_found: "AN",
  };

  const caddRank = {
    P20: { key: "P20" },
    P10: { key: "P10" },
    P0: { key: "P0" },
  };

  const alphamissenseRank = {
    likely_pathogenic: { key: "LP" },
    ambiguous: { key: "AMBIGUOUS" },
    likely_benign: { key: "LB" },
  };

  const siftRank = {
    deleterious: { key: "D" },
    tolerated: { key: "T" },
  };

  const polyphenRank = {
    probably_damaging: { key: "PROBD" },
    possibly_damaging: { key: "POSSD" },
    benign: { key: "B" },
  };

  const fractionDigits3 = new Intl.NumberFormat("en", {
    minimumFractionDigits: 3,
    maximumFractionDigits: 3,
  });

  const escapeAttribute = (value) => {
    return String(value)
      .replace(/&/g, "&amp;")
      .replace(/"/g, "&quot;")
      .replace(/'/g, "&#39;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;");
  };

  const hasNumericValue = (value) => {
    return (
      value !== undefined &&
      value !== null &&
      value !== "" &&
      !Number.isNaN(Number(value))
    );
  };

  const firstValue = (...values) => {
    return values.find(
      (value) => value !== undefined && value !== null && value !== "",
    );
  };

  const toNumericValue = (value) => {
    return Number.parseFloat(String(value));
  };

  const truncateDecimal = (value, digits) => {
    const factor = 10 ** digits;
    return Math.trunc(value * factor) / factor;
  };

  const frequencyFormatter = new Intl.NumberFormat("en", {
    minimumFractionDigits: 4,
    maximumFractionDigits: 4,
  });

  const formatDbSnpLinks = (values) => {
    return (values || [])
      .filter((id) => String(id).match(/^rs\d+$/))
      .map((id) => {
        const rsId = String(id);
        return `<a class="hyper-text -external dbsnp-link" href="https://identifiers.org/dbsnp/${encodeURIComponent(rsId)}" target="_blank" rel="noopener noreferrer">${escapeAttribute(rsId)}</a>`;
      })
      .join(", ");
  };

  const formatRefAlt = (sequence) => {
    const value = sequence || "";
    return value.substring(0, 4) + (value.length > 4 ? "..." : "");
  };

  const resolveFrequencyLevel = (alleleCount, alleleFrequency) => {
    if (Number.isNaN(alleleCount)) return "na";
    if (alleleCount === 0) return "monomorphic";
    if (alleleFrequency < 0.0001) return "<0.0001";
    if (alleleFrequency < 0.001) return "<0.001";
    if (alleleFrequency < 0.01) return "<0.01";
    if (alleleFrequency < 0.05) return "<0.05";
    if (alleleFrequency < 0.5) return "<0.5";
    if (alleleFrequency >= 0.5) return "≥0.5";
    return "na";
  };

  const formatFrequencyDisplayValue = (value) => {
    const numeric = toNumericValue(value);
    if (!Number.isFinite(numeric)) return "";
    if (numeric > 0 && numeric < 0.0001) return "<0.0001";
    return frequencyFormatter.format(truncateDecimal(numeric, 4));
  };

  const buildFrequencyDisplay = (count, value) => {
    const alleleCount = toNumericValue(count);
    const alleleFrequency = toNumericValue(value);

    return {
      frequency: formatFrequencyDisplayValue(value),
      count: count,
      level: resolveFrequencyLevel(alleleCount, alleleFrequency),
    };
  };

  const buildFrequencyMarkerState = (entry) => {
    const altAltCount = firstValue(
      entry.aac,
      entry.allele?.aac,
      entry.homozygote?.aac,
    );
    const hemiAltCount = firstValue(
      entry.hac,
      entry.allele?.hac,
      entry.hemizygote?.hac,
    );
    const hemiRefCount = firstValue(
      entry.hrc,
      entry.allele?.hrc,
      entry.hemizygote?.hrc,
    );
    const hemiOtherAltCount = firstValue(
      entry.hoc,
      entry.allele?.hoc,
      entry.hemizygote?.hoc,
    );

    return {
      has_homozygote_marker: Number(altAltCount) >= 1,
      has_hemizygote_marker: Number(hemiAltCount) >= 1,
      has_hemizygote_value:
        hasNumericValue(hemiAltCount) ||
        hasNumericValue(hemiRefCount) ||
        hasNumericValue(hemiOtherAltCount),
    };
  };

  const formatFrequencyGraph = (frequencies) => {
    const datasets = [
      "gem_j_wga",
      "jga_wgs",
      "jga_wes",
      "jga_snp",
      "tommo",
      "tommo_jsv1",
      "ncbn",
      "bbj1k",
      "bbj2k",
      "jogo",
      "gnomad_genomes",
      "gnomad_exomes",
    ];
    const frequencyBySource = {};

    for (const frequency of frequencies || []) {
      if (frequency.source && !frequencyBySource[frequency.source]) {
        frequencyBySource[frequency.source] = frequency;
      }
    }

    return `<div class="frequency-graph">${datasets
      .map((dataset) => {
        const frequency = frequencyBySource[dataset] || {};
        const af = frequency.af ?? frequency.allele?.frequency;
        const alleleCount = frequency.ac ?? frequency.allele?.count ?? "";
        const display = buildFrequencyDisplay(alleleCount, af);
        const markerState = buildFrequencyMarkerState(frequency);
        const markers = [
          markerState.has_homozygote_marker
            ? `<span class="marker homozygote-marker"></span>`
            : "",
          markerState.has_hemizygote_marker
            ? `<span class="marker hemizygote-marker"></span>`
            : "",
        ].join("");
        return `<div class="dataset" data-dataset="${escapeAttribute(dataset)}" data-frequency="${escapeAttribute(display.level)}" data-frequency-value="${escapeAttribute(display.frequency)}" data-allele-count="${escapeAttribute(display.count ?? "")}" data-has-homozygote-marker="${markerState.has_homozygote_marker}" data-has-hemizygote-marker="${markerState.has_hemizygote_marker}" data-has-hemizygote-value="${markerState.has_hemizygote_value}">${markers}</div>`;
      })
      .join("")}</div>`;
  };

  const formatConsequences = (values) => {
    const items = values
      .map((id) => {
        const consequence = consequenceLabel[id] || {};
        return {
          label: consequence.label || id,
          order: consequence.order || 999,
        };
      })
      .sort((a, b) => a.order - b.order);

    if (items.length === 0) {
      return "";
    }

    const remains = items.length - 1;
    const remainsLabel = remains > 0 ? `${remains}+` : "";
    const consequenceLabels = items.map((item) => item.label);
    return `<div class="remains-content" data-consequences="${escapeAttribute(JSON.stringify(consequenceLabels))}"><div class="consequence-item">${escapeAttribute(items[0].label)}</div><span class="remains-badge" data-remains="${remains}">${remainsLabel}</span></div>`;
  };

  const formatSignificance = (values) => {
    return (values || [])
      .flatMap((item) => item.interpretations || [])
      .map((interpretation) => {
        const value = String(interpretation || "");
        const key = clinicalSignificanceLabel[value]
          ? value
          : clinicalSignificanceKey[value.toLowerCase()];
        const label = clinicalSignificanceLabel[key] || value;
        if (!label) {
          return "";
        }
        return `<span class="clinical-significance-full" data-sign="${escapeAttribute(key || "")}">${escapeAttribute(label)}</span>`;
      })
      .filter(Boolean)
      .join("<br>");
  };

  const formatRankedScore = (value, resolveRank) => {
    const numericValue = toNumericValue(value);
    if (Number.isNaN(numericValue)) {
      return "";
    }

    const rank = resolveRank(numericValue);
    return `<span class="variant-function" data-function="${rank.key}">${fractionDigits3.format(numericValue)}</span>`;
  };

  const formatCadd = (value) =>
    formatRankedScore(value, (numericValue) =>
      numericValue >= 20
        ? caddRank.P20
        : numericValue >= 10
          ? caddRank.P10
          : caddRank.P0,
    );

  const formatAlphamissense = (value) =>
    formatRankedScore(value, (numericValue) =>
      numericValue > 0.564
        ? alphamissenseRank.likely_pathogenic
        : numericValue >= 0.34
          ? alphamissenseRank.ambiguous
          : alphamissenseRank.likely_benign,
    );

  const formatSift = (value) =>
    formatRankedScore(value, (numericValue) =>
      numericValue >= 0.05 ? siftRank.tolerated : siftRank.deleterious,
    );

  const formatPolyphen = (value) =>
    formatRankedScore(value, (numericValue) =>
      numericValue > 0.908
        ? polyphenRank.probably_damaging
        : numericValue > 0.446
          ? polyphenRank.possibly_damaging
          : polyphenRank.benign,
    );

  const formatSscvDb = (values, links) => {
    return (values || [])
      .map((item, index) => {
        const label = item?.predicted_splicing_type;
        const xref = links?.[index]?.xref;
        if (!label) {
          return "";
        }
        const nowrapLabel = escapeAttribute(label).replace(/ /g, "&nbsp;");
        return xref
          ? `<a class="hyper-text -external sscv-link" href="${escapeAttribute(xref)}" target="_blank" rel="noopener noreferrer">${nowrapLabel}</a>`
          : nowrapLabel;
      })
      .filter(Boolean)
      .join(", ");
  };

  return data.flatMap((variant) => {
    try {
      const variantId =
        variant.id ||
        (variant.chromosome &&
        variant.position &&
        variant.reference &&
        variant.alternate
          ? `${variant.chromosome}-${variant.position}-${variant.reference}-${variant.alternate}`
          : "");
      const dbsnp = formatDbSnpLinks(variant.existing_variations);
      const symbols = (variant.genes || [])
        .map((gene) => gene.name)
        .filter(Boolean)
        .map(escapeAttribute)
        .join(", ");
      const frequencies = formatFrequencyGraph(variant.frequencies || []);
      const consequences = [];
      const ref = formatRefAlt(variant.reference);
      const alt = formatRefAlt(variant.alternate);

      if (variant.most_severe_consequence) {
        consequences.push(variant.most_severe_consequence);
      }

      for (const transcript of variant.transcripts || []) {
        for (const consequence of transcript.consequence || []) {
          if (!consequences.includes(consequence)) {
            consequences.push(consequence);
          }
        }
      }

      const significance = formatSignificance(variant.significance);
      const sscvDb = formatSscvDb(
        variant.sscv_db,
        variant.external_links?.sscv_db,
      );

      return [
        {
          report: variantId
            ? `<a class="report-link" href="/variant/${encodeURIComponent(variantId)}"><span class="report-icon"></span></a>`
            : "",
          id: escapeAttribute(variant.id || ""),
          dbsnp: dbsnp,
          position:
            variant.chromosome && variant.position
              ? `<div class="chromosome-position"><div class="chromosome">${escapeAttribute(variant.chromosome)}</div><div class="coordinate">${escapeAttribute(variant.position)}</div></div>`
              : "",
          ref_alt: `<div class="ref-alt"><span class="ref" data-sum="${variant.reference?.length || 0}">${escapeAttribute(ref)}</span><span class="arrow"></span><span class="alt" data-sum="${variant.alternate?.length || 0}">${escapeAttribute(alt)}</span></div>`,
          type: `<span class="variant-type">${escapeAttribute(typeLabel[variant.type] || variant.type || "")}</span>`,
          symbols: symbols,
          frequencies: frequencies,
          consequence: formatConsequences(consequences),
          significance: significance,
          cadd_phred: formatCadd(variant.cadd_phred),
          alphamissense: formatAlphamissense(variant.alphamissense),
          sift: formatSift(variant.sift),
          polyphen: formatPolyphen(variant.polyphen),
          sscv_db: sscvDb,
        },
      ];
    } catch (error) {
      console.error("Failed to format variant", {
        variantId: variant?.id,
        error,
      });
      return [];
    }
  });
};
```
