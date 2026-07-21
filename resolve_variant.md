# Variant IRI from tgv ID or VCF notation

## Parameters

* `tgv_id` TogoVar ID
  * example: tgv47264307
* `variant` VCF representation (CHROM-POS-REF-ALT)
  * example: 12-111803962-G-A

## `tgv_id`

```javascript
async ({SPARQLIST_TOGOVAR_SPARQLIST, SPARQLIST_TOGOVAR_REFERENCE, variant, tgv_id}) => {
  if (tgv_id.length > 0) {
    const url = SPARQLIST_TOGOVAR_SPARQLIST.concat(`/api/tgv2variant?tgv_id=${encodeURIComponent(tgv_id)}`);
    const res = await fetch(url);
    const result = await res.text();

    if (!res.ok) {
      throw new Error(result.replace(/^Error: /, ""));
    }
    
    if (result === "Not found") {
      throw new Error("Not found");
    }

    return result;
  }

  if (variant.length > 0) {
    const regex = /^(chr)?(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)[-:](?<pos>\d+)[-:](?<ref>.+)[->](?<alt>.+)$/;
    const match = variant.match(regex);

    if (match && match.groups) {
      return `http://identifiers.org/hco/${match.groups.chr}/${SPARQLIST_TOGOVAR_REFERENCE}#${match.groups.pos}-${match.groups.ref}-${match.groups.alt}`;
    } else {
      throw new Error(`Invalid notation: ${variant}`);
    }
  }

  throw new Error("Either tgv_id or variant must be provided.");
}
```
