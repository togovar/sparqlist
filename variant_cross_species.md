# Variant report / Cross species

## Parameters

- `tgv_id` TogoVar ID
  - example: tgv47263274
- `variant` VCF representation (CHROM-POS-REF-ALT)
  - example: 12-111774326-G-A

## `variant`

```javascript
async ({ SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id }) => {
  if (tgv_id.length == 0 && variant.length == 0) {
    return "";
  }

  let params;
  if (tgv_id.length > 0) {
    params = `tgv_id=${encodeURIComponent(tgv_id)}`;
  } else {
    params = `variant=${encodeURIComponent(variant)}`;
  }

  const res = await fetch(
    SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/resolve_variant?${params}`),
  );

  if (!res.ok) {
    throw new Error((await res.text()).replace(/^Error: /, ""));
  }

  return await res.text();
};
```

## `result`

```javascript
async ({ SPARQLIST_TOGOVAR_SPARQLIST, variant }) => {
  const mogplus_ver = "mogplus21";
  const regex =
    /^http:\/\/identifiers\.org\/hco\/(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)\/(?<source>GRCh38)#(?<pos>\d+)-(?<ref>.+)-(?<alt>.+)$/;
  const match = variant.match(regex);

  if (!match || !match.groups) {
    throw new Error(`Invalid variant IRI: ${variant}`);
  }

  const params = [
    `source=${encodeURIComponent(match.groups.source)}`,
    `chr=${encodeURIComponent(match.groups.chr)}`,
    `pos=${encodeURIComponent(match.groups.pos)}`,
    `ref=${encodeURIComponent(match.groups.ref)}`,
    `alt=${encodeURIComponent(match.groups.alt)}`,
    `mogplus_ver=${encodeURIComponent(mogplus_ver)}`,
  ].join("&");

  const res = await fetch(
    SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/variant_mogplus?${params}`),
  );

  if (!res.ok) {
    throw new Error((await res.text()).replace(/^Error: /, ""));
  }

  const data = await res.json();

  if (data.error) {
    return [];
  }

  const chrStart = Number(data.pos) - 500;
  const chrEnd = Number(data.pos) + 500;
  const strains = data.strains || [];
  const strainParams = ["refGenome"].concat(
    strains.map((strain) => strain.replace(/\//g, "_")),
  );
  const query = [
    strainParams
      .map((strain) => `strainNoSlct=${encodeURIComponent(strain)}`)
      .join("&"),
    `chrName=${encodeURIComponent(data.chr)}`,
    `chrStart=${chrStart}`,
    `chrEnd=${chrEnd}`,
    "seqType=genome",
    `chrName=${encodeURIComponent(data.chr)}`,
    "geneNameSearchText=",
    "index=submit",
    "presentType=disp",
  ].join("&");
  const url = `https://molossinus.brc.riken.jp/${mogplus_ver}/variantTable/?${query}`;

  return [
    {
      species: "Mouse",
      ref_genome: data.target,
      chr_pos_ref_alt: `${data.chr}-${data.pos}-${data.ref}-${data.alt}`,
      source: "MoG+",
      source_url: url,
      remark: `strains=${strains.join(",")}`,
    },
  ];
};
```
