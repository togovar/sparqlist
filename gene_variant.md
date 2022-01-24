# Gene / variant

## Parameters


* `gene_symbol` gene_symbol
  * default: ALDH2
* `ep` Endpoint
  * default: https://togovar.biosciencedbc.jp/sparql
* `search_api` Search endpoint
  * default: https://togovar.biosciencedbc.jp/search


## `result`

```javascript
async ({search_api, gene_symbol}) => {
  let binding = gene_symbol;

  if (binding) {
    let res = await fetch(search_api.concat("?stat=0&quality=0&term=", binding));
    return res.json();
  } else {
    return { data: [] };
  }
}
```
