# Gene report / Publication

Generate gene2pubmed table data by dbSNP ID

## Parameters

* `hgnc_id`
  * default:
  * example: 404 (ALDH2, 1,176 entries), 1101 (BRCA2, 2,533 entries), 11998 (TP53, 12,167 entries), 555555 (no pmids are found),

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
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dct:  <http://purl.org/dc/terms/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>
PREFIX foaf: <http://xmlns.com/foaf/0.1/>
PREFIX oa:   <http://www.w3.org/ns/oa#>
PREFIX olo:  <http://purl.org/ontology/olo/core#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/vocabulary/>

SELECT DISTINCT ?rs_id ?pmid ?title ?year ?authors ?journal
WHERE {
  VALUES ?ens_gene { {{#each ensembl_gene}} ensg:{{this}} {{/each}} }

  GRAPH <http://togovar.org/variant/annotation/ensembl> {
    ?ens_gene ^tgvo:gene/^tgvo:hasConsequence/rdfs:seeAlso ?rs_id.
  }

  GRAPH <http://togovar.org/pubtator> {
    ?pubtator_node oa:hasBody ?rs_id;
                   a oa:Annotation;
                   oa:hasTarget ?pmid_uri.
  }

  GRAPH <http://togovar.org/pubmed> {
    ?pmid_uri dct:source ?journal;
              dct:title ?title;
              dct:issued ?year;
              bibo:pmid ?pmid;
              dct:creator ?creatorList.

    ?creatorList a olo:OrderedList;
                 olo:length ?length;
                 olo:slot ?slot.

    ?slot olo:index 1;
          olo:item ?person.

    ?person foaf:name ?author.

    BIND(IF(?length > 1, CONCAT(?author, " et al."), ?author) AS ?authors)
  }
}
ORDER BY DESC(?year)
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
        author: x.authors.value,
        journal: x.journal.value
      };
    }else{
      if (ref[x.pmid.value]["rs_id"].includes(x.rs_id.value) == false){
        ref[x.pmid.value]["rs_id"].push(x.rs_id.value);
      }
    }
  })

  return ref;
}
```

## `pmids_all` Concat PMIDs from Pubtator

```javascript
({bib_pubtator}) => {
 
  const pmids_all = [...new Set([...Object.keys(bib_pubtator)])];

  if (pmids_all.length == 0){
    pmids_all.push("no data");
  }

  return pmids_all;
}
```

## `colil` PubMed IDs to their citation count

```javascript
async ({SPARQLIST_TOGOVAR_APP, pmids_all}) => {
  let pmids = pmids_all;
  const PMIDS_PER_REQUEST = 300;

  let results = {};

  return results;

  while(pmids.length > 0){
    let pmids_per_request =
      encodeURIComponent(pmids.splice(0, Math.min(PMIDS_PER_REQUEST, pmids.length)).map(num => String(num)).join(','));

    const sparqlist = ("/sparqlist/api/colil?pmids=").concat(pmids_per_request);
    const res = await fetch(sparqlist, {method: "get", headers: {"Accept": "application/json"}}).then(res => res.json());

    results = { ...results, ...res };
  }

  return results;
}
```

## `result` Compile results

```javascript
({pmids_all, pmid2rsid_litvar, bib_pubtator, colil}) => {
  const articles = [];

  for (const pmid of Object.keys(bib_pubtator)) {
    const href_pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    const href_pubtator = pmid in bib_pubtator ? "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/pubtator3/publication/" + pmid + "\">PubTator3</a>)" : "";

    let links = href_pubmed;
    links += href_pubtator;

    let html = "";
    html += "<b>" + bib_pubtator[pmid].title + "</b><br>\n";
    html += bib_pubtator[pmid].author + "<br>\n";
    html += "<i><b>" + bib_pubtator[pmid].journal + "</b></i><br>\n";

    articles.push([
      links,
      html,
      bib_pubtator[pmid].year.split(" ")[0],
      "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\" >" + (colil[pmid] == undefined ? 0 : colil[pmid]) + "</a>"
    ]);
  }

  return {
    columns: [["PMID"], ["Reference"], ["Year"], ["Cited by"]],
    data: articles
  };
}
```
