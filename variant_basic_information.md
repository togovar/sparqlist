# Variant basic information

## Parameters

* `tgv_id` TogoVar ID
  * default: 48140310

## Endpoint

http://togovar.l5dev.jp/sparql-test

## `result` fetch basic information

```javascript
async ({tgv_id}) => {
  const options = {
    method: 'GET',
    headers: { 'Accept': 'application/json' }
  };

  try{
    var req = []

    req[0] = fetch('http://togovar.l5dev.jp/sparqlist/api/variant_basic_information_1?tgv_id=' + tgv_id, options).then(res => res.json());
    req[1] = fetch('http://togovar.l5dev.jp/sparqlist/api/variant_basic_information_2?tgv_id=' + tgv_id, options).then(res => res.json());
    req[2] = fetch('http://togovar.l5dev.jp/sparqlist/api/variant_basic_information_3?tgv_id=' + tgv_id, options).then(res => res.json());

    var data = Promise.all(req);

    return data.then(function(res){
      return res[0].concat(res[1], res[2]);
    });
  }catch(error){
    console.log(error);
  }
};
```
