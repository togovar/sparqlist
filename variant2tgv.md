# Convert VCF representation to TogoVar ID

## Parameters

* `variant` VCF representation (CHROM-POS-REF-ALT)
  * default: 16-89196249-G-A

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_APP, variant}) => {
  const regex = /^(chr)?(?<chr>[1-9]|1[0-9]|2[0-2]|X|Y|MT?)-(?<pos>\d+)-(?<ref>.+)-(?<alt>.+)/;
  const match = variant.match(regex);

  if (match && match.groups) {
    const url = SPARQLIST_TOGOVAR_APP.concat(`/variant/${match.groups.chr}-${match.groups.pos}-${match.groups.ref}-${match.groups.alt}`);
    const json = await fetch(url, {
      method: 'GET',
      headers: {
        Accept: 'application/json'
      }
    }).then(res => res.json());

    if (json.result && json.result[0]) {
      return json.result[0];
    }
  }

  return 'not found';
}
```
