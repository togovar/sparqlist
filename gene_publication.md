# Gene report / Publication

Generate gene2pubmed table data by dbSNP ID

## Parameters

* `hgnc_id`
  * default: 404
  * example: 28706 (more than one rsids for one pmid), 555555 (no pmids are found)

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `validated_hgnc_id` Validate HGNC_ID

```javascript
({hgnc_id}) => {
  if (hgnc_id.match(/^\d+$/)) {
    return hgnc_id
  }
}
```

## `xref` Get ENSG_ID from HGNC_ID 

```sparql
PREFIX hgnc: <http://identifiers.org/hgnc/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

SELECT DISTINCT ?xref
WHERE {
  {{#if validated_hgnc_id}}
  VALUES ?hgnc_uri { hgnc:{{validated_hgnc_id}} }

  GRAPH <http://togovar.org/hgnc> {
    ?hgnc_uri rdfs:seeAlso ?xref .    
    FILTER STRSTARTS(STR(?xref), "http://identifiers.org/ensembl/")
  }
  {{/if}}
}
```

## `ensembl_gene` Get ENSG_ID from ENSG URI

```javascript
({xref}) => {
  return xref.results.bindings.map(x => x.xref.value.replace("http://identifiers.org/ensembl/", ""));
}
```

## `pubtator_sparql` Get bibliography from PubMed for articles identified by PubTator 

```sparql
DEFINE sql:select-option "order"

PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX oa:   <http://www.w3.org/ns/oa#>
PREFIX olo:  <http://purl.org/ontology/olo/core#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?rs_id ?pmid ?title ?year ?author ?journal
WHERE {
  VALUES ?ens_gene { {{#each ensembl_gene}} ensg:{{this}} {{/each}} }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?ens_gene ^tgvo:gene/^tgvo:hasConsequence/rdfs:seeAlso ?rs_id . 
  }

  GRAPH <http://togovar.org/pubtator> {
    ?pubtator_node oa:hasBody ?rs_id ;
      a oa:Annotation ;
      oa:hasTarget ?pmid_uri .
  }

  GRAPH <http://togovar.org/pubmed> {
    ?pmid_uri dct:source ?journal ;
      dct:title ?title ;
      dct:issued ?year ;
      bibo:pmid ?pmid ;
      dct:creator/olo:slot/olo:item/foaf:name ?author .
  }
}
ORDER BY ?pmid
```

## `bib_pubtator` Concatenate authors from Pubtator 

```javascript
({pubtator_sparql}) => {
  const ref = {};

  pubtator_sparql.results.bindings.forEach((x) => {
    if (x.pmid.value in ref == false){
      ref[x.pmid.value] = {
        rs_id: [x.rs_id.value],
        title: x.title.value,
        year: x.year.value,
        author: [x.author.value],
        journal: x.journal.value
      };
    } else{
      if (ref[x.pmid.value]["rs_id"].includes(x.rs_id.value) == false){
        ref[x.pmid.value]["rs_id"].push(x.rs_id.value);
      }
      if (ref[x.pmid.value]["author"].includes(x.author.value) == false){
        ref[x.pmid.value]["author"].push(x.author.value);
      }
    }
  })

  return ref;
}
```

## `pmid2rsid_litvar` Get relationships from rsID to PubMed IDs identified by LitVar

```javascript
async ({pubtator_sparql}) => {
//  Empty rsids to skip request to LitVar
//  const rsids = [...new Set(pubtator_sparql.results.bindings.map(x => x.rs_id.value.replace("http://identifiers.org/dbsnp/", "")))];
  const rsids = [];
  const pmid2rsid = {} 

  for (const rsid of rsids) {
    try {
      const res = await fetch("https://www.ncbi.nlm.nih.gov/research/litvar2-api/variant/get/litvar%40" + rsid + "%23%23/publications?format=json", {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/x-www-form-urlencoded'
      }
      }).then(res => res.json());

      if("pmids" in res == false){ continue; }

      for (let pmid of res.pmids){
       pmid = pmid.toString();
       if(pmid in pmid2rsid == false){
         pmid2rsid[pmid] = [rsid]
       } else {
         pmid2rsid[pmid].push(rsid);
       }
      }
     } catch (error) {
       console.log(rsid)
       console.log(error);
     }
  }

  return pmid2rsid;
};
```

## `pmids_litvar_only` Get PMID from LitVar not from PubTator

```javascript
({pubtator_sparql, pmid2rsid_litvar}) => {
  if (!pmid2rsid_litvar) { return []; }

  pmids_pubtator = pubtator_sparql.results.bindings.map(x => x.pmid.value)
  const pmids_litvar_only = Object.keys(pmid2rsid_litvar).filter(pmid => pmids_pubtator.indexOf(pmid) == -1)

  return pmids_litvar_only
}
```

## `litvar_only_sparql` Get bibliography from PubMed for articles identified by LitVar

```sparql
PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX dct:    <http://purl.org/dc/terms/>
PREFIX olo:    <http://purl.org/ontology/olo/core#>
PREFIX foaf:   <http://xmlns.com/foaf/0.1/>
PREFIX pubmed: <http://rdf.ncbi.nlm.nih.gov/pubmed/>

SELECT DISTINCT ?pmid_uri ?pmid ?title ?year ?author ?journal
WHERE {
  VALUES ?pmid { {{#each pmids_litvar_only}}'{{this}}' {{/each}} }

  GRAPH <http://togovar.org/pubmed> {
    ?pmid_uri bibo:pmid ?pmid ;
      dct:title ?title ;
      dct:issued ?year ;
      dct:creator/olo:slot/olo:item/foaf:name ?author ;
      dct:source ?journal .
  }
}
```

## `bib_litvar_only` Concatenate authors from LitVar 

```javascript
({litvar_only_sparql}) => {
  const ref = {};

  litvar_only_sparql.results.bindings.forEach((x) => {
    if (x.pmid.value in ref == false){
      ref[x.pmid.value] = {
        title: x.title.value,
        year: x.year.value,
        author: [x.author.value],
        journal: x.journal.value
      };
    } else{
      if (ref[x.pmid.value]["author"].includes(x.author.value) == false){
        ref[x.pmid.value]["author"].push(x.author.value);
      }
    }
  })

  return ref;
}
```

## `pmids_all` Concat PMIDs from Pubtator and Litvar

```javascript
({bib_litvar_only, bib_pubtator}) => {
 
  const pmids_all = [...new Set([...Object.keys(bib_litvar_only), ...Object.keys(bib_pubtator)])];

  if(pmids_all.length == 0){
    pmids_all.push("no data");
  }

  return pmids_all;
}
```

## Endpoint

https://colil.dbcls.jp/sparql

## `colil` PubMed IDs to citation count

```sparql
PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX colil:  <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{#each pmids_all}}'{{this}}' {{/each}} }

  GRAPH <http://purl.jp/bio/10/colil/core> {
    ?pmid ^togows:pmid ?pubmed .
    ?pubmed a colil:PubMed ;
      ^rdfs:seeAlso/^bibo:cites ?citation_paper.
  }
}
```

## `result` Compile results

```javascript
({pmids_all, pmid2rsid_litvar, bib_pubtator, bib_litvar_only, colil}) => {
  const articles = [];
  const pubtator_litvar = Object.assign(bib_pubtator, bib_litvar_only)

  for (const pmid of Object.keys(pubtator_litvar)) {
    const href_pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    const href_pubtator = pmid in bib_pubtator ? "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/pubtator?view=publication&pmid=" + pmid + "\">PubTatorCentral</a>)" : "";

    let href_litvar = "";
    if(pmid in pmid2rsid_litvar == true){
      href_litvar = "(Litvar: ";
      href_litvar += pmid2rsid_litvar[pmid].map(
	rsid => "<a href=\"https://www.ncbi.nlm.nih.gov/research/litvar2/publication/" + pmid + "?variant=litvar%40" + rsid + "%23%23\">" + rsid + "</a>"
	).join(' ,')
      href_litvar +=")";
    }  

    let links = href_pubmed;
    links += href_pubtator;
    links += href_litvar;

    let html = "";
    html += "<b>" + pubtator_litvar[pmid].title + "</b><br>\n";
    html += pubtator_litvar[pmid].author.join(" ,") + "<br>\n";
    html += "<i><b>" + pubtator_litvar[pmid].journal + "</b></i><br>\n";

    articles.push([
      links,
      html,
      pubtator_litvar[pmid].year.split(" ")[0],
      "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\" >" + (colil[pmid] == undefined ? 0 : colil[pmid]) + "</a>"
    ]);
  }

  return {
    columns: [["PMID"], ["Reference"], ["Year"], ["Cited by"]],
    data: articles
  };
}
```
