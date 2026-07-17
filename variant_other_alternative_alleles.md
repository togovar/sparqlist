# Variant report / Other alternative alleles

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv30913364
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 7-127614533-G-A

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `variant`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, variant, tgv_id}) => {
  let params;
  if (tgv_id.length > 0) {
    params = `tgv_id=${encodeURIComponent(tgv_id)}`;
  } else if (variant.length > 0) {
    params = `variant=${encodeURIComponent(variant)}`;
  } else {
    throw new Error("Either tgv_id or variant must be provided.");
  }

  const res = await fetch(SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/resolve_variant?${params}`));

  if (!res.ok) {
    throw new Error((await res.text()).replace(/^Error: /, ""));
  }

  return await res.text();
}
```

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_APP, variant}) => {
  const regex = /^http:\/\/identifiers\.org\/hco\/(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)\/GRCh3[78]#(?<pos>\d+)-(?<ref>.+)-(?<alt>.+)$/;
  const match = variant.match(regex);

  if (match && match.groups) {
    let region = `${match.groups.chr}:${match.groups.pos}`;
    if (match.groups.ref.length > 1) {
      region += `-${match.groups.pos + match.groups.ref.length - 1}`;
    }

    const res = await fetch(SPARQLIST_TOGOVAR_APP.concat("/search?stat=0&quality=0&term=", encodeURIComponent(region)), {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
      },
    });

    return res.json();
  } else {
    return {data: []};
  }
}
```
