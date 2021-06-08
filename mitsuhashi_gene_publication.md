# Gene Report publication

## Parameters

* `hgnc_id` HGNC ID
  * default: 404
  * example: 404, 3084, 1101

## Endpoint

https://togovar-dev.biosciencedbc.jp/sparql


### `hgnc2pmids` HGNC ID to PubMed IDs by Pubtator

```sparql

PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX ncbigene: <http://identifiers.org/ncbigene/>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX olo: <http://purl.org/ontology/olo/core#>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
SELECT DISTINCT ?pmid_uri ?pmid ?title ?year ?authors ?journal  
WHERE {
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }
 
  GRAPH <http://togovar.biosciencedbc.jp/hgnc>{ 
    ?hgnc_uri rdfs:seeAlso ?ensg.
    ?ensg dct:identifier ?ensg_id.
    FILTER STRSTARTS(STR(?ensg), "http://identifiers.org/ensembl/ENSG")
  }

  GRAPH <http://togovar.biosciencedbc.jp/variant>{ 
    ?variant tgvo:hasConsequence / tgvo:gene / dct:identifier ?ensg_id.
    ?variant rdfs:seeAlso ?dbsnp.

    FILTER STRSTARTS(STR(?dbsnp), 'http://identifiers.org/dbsnp/').
    BIND (REPLACE(STR(?dbsnp),'http://identifiers.org/dbsnp/','') as ?rsid).
  } 

  GRAPH <http://togovar.biosciencedbc.jp/pubtator>{
    ?pubtator_node rdf:type oa:Annotation ;
       oa:hasBody ?rs_id;
       oa:hasTarget ?pmid_uri.
  }

   {
    #  See https://docs.cambridgesemantics.com/anzograph/v2.2/userdoc/group-concat.htm
    SELECT ?pmid_uri (CONCAT(group_concat(distinct ?author ; separator = ", "), ?et_al) AS ?authors)
    WHERE {
      GRAPH <http://togovar.biosciencedbc.jp/pubmed>{
        ?pmid_uri dcterms:source ?journal.
        ?pmid_uri dcterms:creator ?creator_node.
        ?creator_node olo:length ?num_authors.
        ?creator_node olo:slot ?slot .
        ?slot olo:index ?item_idx.
        ?slot olo:item ?item .
        ?item foaf:name ?author .
        FILTER(?item_idx < 2)
        BIND(IF(?num_authors > 1, " et al.", "") AS ?et_al)
       }
     }
     GROUP BY ?pmid_uri ?et_al
   }
      
    GRAPH <http://togovar.biosciencedbc.jp/pubmed>{
      ?pmid_uri dcterms:source ?journal ;
      dcterms:title ?title ;
      dcterms:issued ?year ;
      bibo:pmid ?pmid .
   }
} ORDER BY DESC(?year) limit 100
```


## `shaping_pmidinfo` Shaping pmid infomation

```javascript
({hgnc2pmids}) => {
  let ref = {}
  hgnc2pmids.results.bindings.forEach((x) => {
    if (ref[x.pmid.value]) {
      ref[x.pmid.value]["author"] = ref[x.pmid.value]["author"] + ", " + x.authors.value
    }else{
      ref[x.pmid.value] = {pmid_uri: x.pmid_uri.value, title: x.title.value, year: x.year.value, author: x.authors.value, journal: x.journal.value.replace("():",":")}
    }
  })
  return ref
}
```

## `pmids`

```javascript
({shaping_pmidinfo}) => {
  return Object.keys(shaping_pmidinfo).filter(function (x, i, self){ return self.indexOf(x) === i;}).map(pmid => '"' + pmid + '"' ).join(" ")

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

SELECT distinct ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{pmids}} }
#  VALUES ?pmid { {{concat_pmids}} }
  ?citation_paper bibo:cites ?reference_paper .
  ?reference_paper rdfs:seeAlso ?dummy .
  ?dummy rdf:type colil:PubMed ;
    togows:pmid ?pmid .
}
```

## `result` Compile results

```javascript
({shaping_pmidinfo, pmid2citation}) =>{
  let articles = {};
  let mesh_lsd = {};
  let pubtator_pmids = Object.keys(shaping_pmidinfo)
  
  
  for (let pmid in shaping_pmidinfo){
    console.log(shaping_pmidinfo[pmid].year);
    let pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    let pubtator = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/pubtator/?view=docsum&query=" + pmid + "\">PubTatorCentral</a>)";
    let pmid_info = pubmed;
    if(pubtator_pmids.includes(pmid)){
      pmid_info += pubtator;
    }
    
    articles[pmid] = {
      pmid: pmid_info ,
      diseases: []
    };
    
    let html = "";
    html += "<b>" + shaping_pmidinfo[pmid].title + "</b><br>\n";
    html += shaping_pmidinfo[pmid].author + "<br>\n";
    html += "<i><b>" + shaping_pmidinfo[pmid].journal + "</b></i><br>\n";
    articles[pmid].reference = html;
    articles[pmid].year = shaping_pmidinfo[pmid].year.split(" ")[0];
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
    columns: [["PMID"], ["Reference"], ["Year"], ["Cited by"]],
    data: pubtator_pmids.map(x => {
      let article = articles[x];
      return [article.pmid, article.reference, article.year, article.citation]
    })
  };
}
```
