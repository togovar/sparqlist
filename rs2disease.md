# TogoVar rs2disease stanza query

Generate rs2disease table data by dbSNP ID

## Note

This is old (depending on external sparqlists). Please use new [English version](https://togovar.org/sparqlist/rs2diseases_en) or [Japanese version](https://togovar.org/sparqlist/rs2diseases_ja)

## Parameters

* `rs` dbSNP ID
  * default: rs4917014
  * examples: rs864309721

## `pmids` PubMed IDs

```javascript
async ({rs}) => {
  const sparqlist = "https://togovar.org/sparqlist/api/"
  const options = { headers: { "Accept": "application/json" } }
  try {
    var rest = sparqlist + "rs2disease_articles?rs=" + rs
    var data = await fetch(rest, options).then(res => res.json())
    var body = data.results.bindings
    if (body) {
      var prefix = "http://identifiers.org/pubmed/"
      return body.map(item => item.article.value.replace(prefix, ""))
    }
  } catch(error) {
    console.log(error)
  }
}
```

## `pmid_mesh` MeSH IDs

```javascript
async ({pmids}) => {
  const sparqlist = "https://togovar.org/sparqlist/api/"
  const options = { headers: { "Accept": "application/json" } }
  try {
    var rest = sparqlist + "rs2disease_annotation?pmids=" + pmids.join(",")
    var data = await fetch(rest, options).then(res => res.json())
    var body = data.results.bindings
    if (body) {
      //var prefix = "http://identifiers.org/mesh/"
      //return body.map(item => item.annotation.value.replace(prefix, "")).join(",")
      return body
    }
  } catch(error) {
    console.log(error)
  }
}
```

## `biblio` Publication info

```javascript
async ({pmids}) => {
  const sparqlist = "https://togovar.org/sparqlist/api/"
  const options = { headers: { "Accept": "application/json" } }
  try {
    var rest = sparqlist + "rs2disease_biblio?pmids=" + pmids.join(",")
    var data = await fetch(rest, options).then(res => res.json())
    var body = data.results.bindings
    if (body) {
      return body
    }
  } catch(error) {
    console.log(error)
  }
}
```

## `citation` Citation info

```javascript
async ({pmids}) => {
  const sparqlist = "https://togovar.org/sparqlist/api/"
  const options = { headers: { "Accept": "application/json" } }
  try {
    var rest = sparqlist + "rs2disease_citation?pmids=" + pmids.join(",")
    var data = await fetch(rest, options).then(res => res.json())
    var body = data.results.bindings
    if (body) {
      return body
    }
  } catch(error) {
    console.log(error)
  }
}
```

## `mesh_labels` MeSH labels

```javascript
async ({pmid_mesh}) => {
  var prefix = "http://identifiers.org/mesh/"
  var meshs = pmid_mesh.map(item => item.annotation.value.replace(prefix, ""))

  const sparqlist = "https://togovar.org/sparqlist/api/"
  const options = { headers: { "Accept": "application/json" } }
  try {
    var rest = sparqlist + "rs2disease_meshlabel?meshs=" + meshs.join(",")
    var data = await fetch(rest, options).then(res => res.json())
    var body = data.results.bindings
    if (body) {
      return body
    }
  } catch(error) {
    console.log(error)
  }
}
```

## `results` rs# to pmid#

```javascript
({pmids, biblio, citation, mesh_labels, pmid_mesh}) => {
  var articles = {}
  var mesh_lsd = {}
  pmids.forEach(pmid => {
    var pubtator = " (<a href=\"https://www.ncbi.nlm.nih.gov/CBBresearch/Lu/Demo/PubTator/curator_identifier.cgi?pmid=" + pmid + "&Gene_display=1&Disease_display=1&Mutation_display=1&Species_display=1&Chemical_display=1\">PubTator</a>)"
    articles[pmid] = {
      "pmid": "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>" + pubtator,
      "diseases": []
    }
  })
  biblio.forEach(item => {
    var pmid = item.pmid.value
    var html = ""
    html += "<b>" + item.article_title.value + "</b><br>\n"
    html += item.authors.value + "<br>\n"
    html += "<i><b>" + item.source.value + "</b></i><br>\n"
    articles[pmid].reference = html
    articles[pmid].year = item.year.value
  })
  citation.forEach(item => {
    var pmid = item.pmid.value
    articles[pmid].citation = "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + item.citation_count.value + "</a>"
  })
  
  mesh_labels.forEach(item => {
    var prefix = "http://purl.jp/bio/10/lsd/mesh/"
    var mesh = item.mesh.value.replace(prefix, "")
    mesh_lsd[mesh] = {}
    mesh_lsd[mesh].ja = item.ja_label.value || null
    mesh_lsd[mesh].en = item.en_label.value || null
  })
  pmid_mesh.forEach(item => {
    var prefix = "http://identifiers.org/pubmed/"
    var pmid = item.article.value.replace(prefix, "")
    var prefix = "http://identifiers.org/mesh/"
    var mesh = item.annotation.value.replace(prefix, "")
    var disease = "<a href=\"https://www.ncbi.nlm.nih.gov/mesh/?term="  + mesh +  "\">" + "MeSH:" + mesh + "</a>"
    if (mesh_lsd[mesh]) {
      disease += " " + mesh_lsd[mesh].en
      //disease += " (" + mesh_lsd[mesh].ja + ")"
    } else {
      disease += " (not found in LSD)"
    }
    articles[pmid].diseases.push(disease)
  })

  var results = {
    "header": [ "PMID", "Reference", "Year", "Cited by", "Diseases" ],
    "data": []
  }
  
  pmids.forEach(pmid => {
    var article = articles[pmid]
    results.data.push(
      [ article.pmid, article.reference, article.year, article.citation, article.diseases ]
    )
  })

  return results
}
```
