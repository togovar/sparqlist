# Gene report / Variant

## Parameters

* `hgnc_id` HGNC ID
  * default: 404
* `offset` number of skip records
  * default: 0

## `result`

```javascript
async ({SPARQLIST_TOGOVAR_API, hgnc_id, offset}) => {
  const RECORDS_PER_PAGE = 100;
  const OFFSET_MAX = 10000;

  offset = parseInt(offset) || 0;

  if (offset + RECORDS_PER_PAGE <= OFFSET_MAX) {
    const query = {
      formatter: 'html',
      query: {
        gene: {
          relation: 'eq',
          terms: [
            parseInt(hgnc_id)
          ]
        }
      },
      offset: offset,
      limit: RECORDS_PER_PAGE
    };

    return await fetch(SPARQLIST_TOGOVAR_API.concat('/search/variant.json'), {
      method: 'POST',
      headers: {
        Accept: 'application/json',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(query)
    }).then(res => res.json());
  } else {
    return [];
  }
}
```
