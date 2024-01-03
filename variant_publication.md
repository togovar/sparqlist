# TogoVar variant_publication stanza query

Generate rs2pubmed table data by dbSNP ID

## Parameters

* `rs` dbSNP ID
  * default: rs671
  * example: rs671(hit both), rs797044836(pubTatorCentral only), rs112750067(no hits)

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `rs2pmid` dbSNP ID to PubMed Info by Pubtator and PubMed

```sparql
DEFINE sql:select-option "order"

PREFIX bibo:  <http://purl.org/ontology/bibo/>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>
PREFIX dct:   <http://purl.org/dc/terms/>
PREFIX foaf:  <http://xmlns.com/foaf/0.1/>
PREFIX oa:    <http://www.w3.org/ns/oa#>
PREFIX olo:   <http://purl.org/ontology/olo/core#>

SELECT DISTINCT ?pmid_uri ?pmid ?title ?year ?author ?journal
WHERE {
  GRAPH <http://togovar.org/pubtator> {
    dbsnp:{{rs}} ^oa:hasBody ?pubtator_node .

    ?pubtator_node a oa:Annotation ;
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
```

## `shaping_pmidinfo` Shaping pmid infomation

```javascript
({rs2pmid}) => {
  const ref = {};

  rs2pmid.results.bindings.forEach(x => {
    if (ref[x.pmid.value]) {
      ref[x.pmid.value]["author"] = ref[x.pmid.value]["author"] + ", " + x.author.value
    } else {
      ref[x.pmid.value] = {
        pmid_uri: x.pmid_uri.value,
        title: x.title.value,
        year: x.year.value,
        author: x.author.value,
        journal: x.journal.value
      }
    }
  })

  return ref
}
```

## `rs2pmid_litvar` dbSNP ID to PubMed IDs by Litvar

```javascript
async ({rs}) => {
  const param = "?query=%5B%22litvar%40" + rs + "%23%23%22%5D";

  try {
    const res = await fetch("https://www.ncbi.nlm.nih.gov/research/bionlp/litvar/api/v1/public/pmids" + param, {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    }).then(res => res.json());

    return res.map(x => x.pmid.toString());
  } catch (error) {
    console.log(error);
  }
};
```

## `dup_pmid_litvar` Remove duplicate PMID

```javascript
({rs2pmid_litvar, shaping_pmidinfo}) => {
  const ret = rs2pmid_litvar.filter(i => Object.keys(shaping_pmidinfo).indexOf(i) == -1)

  if (ret.length > 0) {
    return ret.map(x => x.replace(/^/, "pubmed:")).join(" ")
  } else {
    return "'nodata'"
  }
}
```

## `concat_pmids` Concat PMIDs from Pubtator and Litvar

```javascript
({rs2pmid_litvar, shaping_pmidinfo}) => {
  const ret = rs2pmid_litvar.concat(Object.keys(shaping_pmidinfo)).filter((x, i, self) => self.indexOf(x) === i);

  if (ret.length > 0) {
    return ret.map(pmid => '"' + pmid + '"').join(" ")
  } else {
    return "'nodata'"
  }
}
```

## `litvar2pmidinfo` PMIDs to infomation

```sparql
PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX dct:    <http://purl.org/dc/terms/>
PREFIX olo:    <http://purl.org/ontology/olo/core#>
PREFIX foaf:   <http://xmlns.com/foaf/0.1/>
PREFIX pubmed: <http://rdf.ncbi.nlm.nih.gov/pubmed/>

SELECT DISTINCT ?pmid_uri ?pmid ?title ?year ?author ?journal
WHERE {
  VALUES ?pmid_uri { {{dup_pmid_litvar}} }

  GRAPH <http://togovar.org/pubmed> {
    ?pmid_uri bibo:pmid ?pmid ;
      dct:title ?title ;
      dct:issued ?year ;
      dct:creator/olo:slot/olo:item/foaf:name ?author ;
      dct:source ?journal .
  }
}
```

## Endpoint

https://colil.dbcls.jp/sparql

## `pmid2citation` PubMed IDs to citation count

```sparql
DEFINE sql:select-option "order"

PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX colil:  <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{concat_pmids}} }

  GRAPH <http://purl.jp/bio/10/colil/core> {
    ?pmid ^togows:pmid ?pubmed .
    ?pubmed a colil:PubMed ;
      ^rdfs:seeAlso/^bibo:cites ?citation_paper.
  }
}
```

## `shaping_pmidinfo_litvar` Shaping PMIDs infomation

```javascript
({litvar2pmidinfo}) => {
  const ref = {}

  litvar2pmidinfo.results.bindings.forEach(x => {
    if (ref[x.pmid.value]) {
      ref[x.pmid.value]["author"] = ref[x.pmid.value]["author"] + ", " + x.author.value
    } else {
      ref[x.pmid.value] = {
        pmid_uri: x.pmid_uri.value,
        title: x.title.value,
        year: x.year.value,
        author: x.author.value,
        journal: x.journal.value
      }
    }
  })

  return ref
}
```

## `result` Compile results

```javascript
({rs, rs2pmid_litvar, shaping_pmidinfo, shaping_pmidinfo_litvar, pmid2citation}) => {
  const articles = {};
  const ordered_pmids = Object.keys(shaping_pmidinfo).concat(Object.keys(shaping_pmidinfo_litvar)).sort()
  const pubtator_pmids = Object.keys(shaping_pmidinfo)
  const pmids_info = Object.assign(shaping_pmidinfo, shaping_pmidinfo_litvar)

  for (let pmid in pmids_info) {
    const pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    const pubtator = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/pubtator?view=docsum&query=" + pmid + "\">PubTatorCentral</a>)";
    const litvar = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/litvar2/docsum?variant=litvar@" + rs + "%23%23" + "\">Litvar</a>)";
    let pmid_info = pubmed;

    if (pubtator_pmids.includes(pmid)) {
      pmid_info += pubtator;
    }
    if (rs2pmid_litvar.includes(pmid)) {
      pmid_info += litvar;
    }

    articles[pmid] = {
      pmid: pmid_info,
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
  }

  pmid2citation.results.bindings.forEach(x => {
    const pmid = x.pmid.value;
    if (articles[pmid]) {
      articles[pmid].citation = "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + x.citation_count.value + "</a>";
    }
  });

  return {
    columns: [["PMID"], ["Reference"], ["Year"], ["Cited by"]],
    data: ordered_pmids.map(x => {
      const article = articles[x];

      return [article.pmid, article.reference, article.year, article.citation]
    })
  };
}
```
