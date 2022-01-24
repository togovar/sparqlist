# TogoVar rs2disease stanza query

Generate rs2disease table data by dbSNP ID

## Parameters

* `rs` dbSNP ID
  * example: rs671, rs4917014 

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `rs2pmid` dbSNP ID to PubMed IDs by Pubtator

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX olo: <http://purl.org/ontology/olo/core#>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
SELECT ?pmid_uri  
WHERE {
   GRAPH <http://togovar.biosciencedbc.jp/pubtator>
   { ?pubtator_node rdf:type oa:Annotation ;
      oa:hasTarget ?pmid_uri ;
      oa:hasBody dbsnp:{{rs}} .
   }
}
```

## `change_uri` Modify URI

```javascript
({rs2pmid}) => {
  let prefix = "http://identifiers.org/pubmed/";
  return rs2pmid.results.bindings.map(x => x.pmid_uri.value.replace(prefix, "pubmed:")).join(" ")
}
```

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `rs2pmid2` PubMed IDs to info

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX olo: <http://purl.org/ontology/olo/core#>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX pubmed: <http://rdf.ncbi.nlm.nih.gov/pubmed/>
SELECT ?pmid_uri  ?pmid ?title ?year ?author ?journal
WHERE {
   VALUES ?pmid_uri  { {{change_uri}} }
   GRAPH <http://togovar.biosciencedbc.jp/pubmed>
   {     
      ?pmid_uri dcterms:source ?journal ;
      dcterms:creator ?creator_node ;
      dcterms:title ?title ;
      dcterms:issued ?year ;
      bibo:pmid ?pmid .
      ?creator_node olo:slot ?slot .
      ?slot olo:item ?item .
      ?item foaf:name ?author .
   }
}
```

## `shaping_pmidinfo` Shaping pmid infomation

```javascript
({rs2pmid2}) => {
  let ref = {}
  rs2pmid2.results.bindings.forEach((x) => {
    if (ref[x.pmid.value]) {
      ref[x.pmid.value]["author"] = ref[x.pmid.value]["author"] + ", " + x.author.value
    }else{
      ref[x.pmid.value] = {pmid_uri: x.pmid_uri.value, title: x.title.value, year: x.year.value, author: x.author.value, journal: x.journal.value}
    }
  })
  return ref
}
```

## `rs2pmid_litvar` dbSNP ID to PubMed IDs by Litvar

```javascript
async ({rs})=>{
    var param = "?query=%5B%22litvar%40" + rs + "%23%23%22%5D"
    const options = {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    };
    try{
      var res = await fetch("https://www.ncbi.nlm.nih.gov/research/bionlp/litvar/api/v1/public/pmids"+ param, options).then(res=>res.json());
      var pmids = [];
      for(var i=0; i < res.length; i++){
        pmids.push(res[i]["pmid"].toString());
      }
      return pmids;
    }catch(error){
      console.log(error);
    }
};
```

## `dup_pmid_litvar` Remove duplicate PMID

```javascript
({rs2pmid_litvar, shaping_pmidinfo}) => {  
  return rs2pmid_litvar.filter(i => Object.keys(shaping_pmidinfo).indexOf(i)== -1).map(x => x.replace(/^/, "pubmed:")).join(" ")
}
```

## `concat_pmids` Concat PMID by Pubtator and Litvar

```javascript
({rs2pmid_litvar, shaping_pmidinfo}) => {  
  return rs2pmid_litvar.concat(Object.keys(shaping_pmidinfo)).filter(function (x, i, self){ return self.indexOf(x) === i;}).map(pmid => '"' + pmid + '"' ).join(" ")
}

```
## Endpoint
http://colil.dbcls.jp/sparql

## `pmid2citation` PubMed IDs to citation count

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX colil: <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{concat_pmids}} }
  ?citation_paper bibo:cites ?reference_paper .
  ?reference_paper rdfs:seeAlso ?dummy .
  ?dummy rdf:type colil:PubMed ;
    togows:pmid ?pmid .
}
```

## Endpoint

https://togovar.biosciencedbc.jp/sparql

## `litvar2pmidinfo` PMIDs to infomation

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX olo: <http://purl.org/ontology/olo/core#>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX pubmed: <http://rdf.ncbi.nlm.nih.gov/pubmed/>
SELECT ?pmid_uri  ?pmid ?title ?year ?author ?journal
WHERE {
   VALUES ?pmid_uri  { {{dup_pmid_litvar}} }
   GRAPH <http://togovar.biosciencedbc.jp/pubmed>
   {     
      ?pmid_uri dcterms:source ?journal ;
      dcterms:creator ?creator_node ;
      dcterms:title ?title ;
      dcterms:issued ?year ;
      bibo:pmid ?pmid .
      ?creator_node olo:slot ?slot .
      ?slot olo:item ?item .
      ?item foaf:name ?author .
   }
}
```

## `shaping_pmidinfo_litvar` Shaping PMIDs infomation

```javascript
({litvar2pmidinfo}) => {
  let ref = {}
  litvar2pmidinfo.results.bindings.forEach((x) => {
    if (ref[x.pmid.value]) {
      ref[x.pmid.value]["author"] = ref[x.pmid.value]["author"] + ", " + x.author.value
    }else{
      ref[x.pmid.value] = {pmid_uri: x.pmid_uri.value, title: x.title.value, year: x.year.value, author: x.author.value, journal: x.journal.value}
    }
  })
  return ref
}
```

## `result` Compile results

```javascript
({rs, rs2pmid_litvar, shaping_pmidinfo, shaping_pmidinfo_litvar,pmid2citation}) =>{
  let articles = {};
  let mesh_lsd = {};
  let ordered_pmids = Object.keys(shaping_pmidinfo).concat(Object.keys(shaping_pmidinfo_litvar)).sort()
  let pubtator_pmids = Object.keys(shaping_pmidinfo)
  let pmids_info = Object.assign(shaping_pmidinfo, shaping_pmidinfo_litvar)
  
  
  for (let pmid in pmids_info){
    console.log(pmids_info[pmid].year);
    let pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    let pubtator = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/pubtator/?view=docsum&query=" + pmid + "\">PubTatorCentral</a>)";
    let litvar = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/CBBresearch/Lu/Demo/LitVar/#!?query="+ rs + "\">Litvar</a>)";
    let pmid_info = pubmed;
    if(pubtator_pmids.includes(pmid)){
      pmid_info += pubtator;
    }
    if(rs2pmid_litvar.includes(pmid)){
      pmid_info += litvar;
    }
    
    articles[pmid] = {
      pmid: pmid_info ,
      diseases: []
    };
    
    let html = "";
    html += "<b>" + pmids_info[pmid].title + "</b><br>\n";
    html += pmids_info[pmid].author + "<br>\n";
    html += "<i><b>" + pmids_info[pmid].journal + "</b></i><br>\n";
    articles[pmid].reference = html;
    articles[pmid].year = pmids_info[pmid].year.split(" ")[0];
    articles[pmid].citation = "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\" >" + 0 + "</a>";
    articles[pmid].diseases = "meshのリンク"
  };
  
  pmid2citation.results.bindings.forEach(x => {
    let pmid = x.pmid.value;
    if (articles[pmid]) {
      articles[pmid].citation = "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + x.citation_count.value + "</a>";      
    }
  });
  
  return {
    columns: [["PMID"], ["Reference"], ["Date"], ["Cited by"]],
    data: ordered_pmids.map(x => {
      let article = articles[x];
      return [article.pmid, article.reference, article.year, article.citation]
    })
  };
}
```
