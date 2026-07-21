# Convert VCF representation to TogoVar ID

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * default: 12-111803962-G-A
  * example:
    * 12:111803962:G>A
    * chr12-111803962-G-A
    * chr12:111803962:G>A

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_APP, variant}) => {
  const regex = /^(chr)?(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)[-:](?<pos>\d+)[-:](?<ref>.+)[->](?<alt>.+)/;
  const match = variant.match(regex);

  if (match && match.groups) {
    const url = SPARQLIST_TOGOVAR_APP.concat("/api/search/variant");
    const json = await fetch(url, {
      method: "POST",
      headers: {
        "Accept": 'application/json',
        "Content-Type": 'application/json',
      },
      body: JSON.stringify({
        "query": {
          "variant": {
            "chromosome": match.groups.chr,
            "position": parseInt(match.groups.pos),
            "reference": match.groups.ref,
            "alternate": match.groups.alt,
          }
        }
      })
    }).then(res => res.json());

    if (json?.data?.[0]?.id) {
      return json.data[0].id;
    }
  }

  return "Not found";
};
```
