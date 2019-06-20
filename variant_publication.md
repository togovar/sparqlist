# TogoVar rs2disease stanza query

Generate rs2disease table data by dbSNP ID

## Parameters

* `rs` dbSNP ID
  * default: rs4917014
  * examples: rs864309721

## Endpoint

http://ep.dbcls.jp/sparql-togovar

### `rs2pmid` dbSNP ID to PubMed IDs

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX dbsnp: <http://identifiers.org/dbsnp/>

SELECT DISTINCT ?pmid_uri
WHERE {
   ?node rdf:type oa:Annotation ;
         oa:hasTarget ?pmid_uri ;
         oa:hasBody dbsnp:{{rs}} .
}
```

#### `pmids` Array of PubMed IDs

```javascript
({rs2pmid}) => {
  let prefix = "http://identifiers.org/pubmed/";
  
  return rs2pmid.results.bindings.map(item => item.pmid_uri.value.replace(prefix, ""));
}
```

#### `pmid_qnames` Joined PubMed ID QNames with the pmid: prefix

```javascript
({pmids}) => {
  return pmids.map(pmid => "pmid:" + pmid).join(" ");
}
```

#### `pmid_values` Joined PubMed ID literals for VALUES

```javascript
({pmids}) => {
  return pmids.map(pmid => '"' + pmid + '"').join(" ");
}
```

### `pmid2reference` PubMed IDs to reference information

```sparql
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>
PREFIX colil: <http://purl.jp/bio/10/colil/ontology/201303#>

SELECT DISTINCT ?pmid ?title ?authors ?journal ?year
WHERE {
  VALUES ?pmid  { {{pmid_values}} }
  ?article colil:Authors ?authors ;
           togows:pmid ?pmid ;
           togows:ti ?title ;
           togows:so ?journal ;
           togows:dp ?year .
}
```

#### `ordered_pmids` Array of PubMed IDs ordered by year

```javascript
({pmid2reference}) => {
  return pmid2reference.results.bindings.map(item => item.pmid.value);
}
```

## Endpoint

http://colil.dbcls.jp/sparql

### `pmid2citation` PubMed IDs to citation count

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX colil: <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{pmid_values}} }
  ?citation_paper bibo:cites ?reference_paper .
  ?reference_paper rdfs:seeAlso ?dummy .
  ?dummy rdf:type colil:PubMed ;
         togows:pmid ?pmid .
}
```

## Endpoint

http://ep.dbcls.jp/sparql-togovar

### `pmid2mesh` PubMed ID to MeSH IDs

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX pmid: <http://identifiers.org/pubmed/>

SELECT ?pmid_uri ?mesh_uri
WHERE {
  VALUES ?pmid_uri { {{pmid_qnames}} }
  ?node rdf:type oa:Annotation ;
        oa:hasTarget ?pmid_uri ;
        oa:hasBody ?mesh_uri .
  FILTER (STRSTARTS(STR(?mesh_uri), "http://identifiers.org/mesh/"))
}
```

#### `meshs` Array of MeSH IDs

```javascript
({pmid2mesh}) => {
  let prefix = "http://identifiers.org/mesh/";
  
  return pmid2mesh.results.bindings.map(item => item.mesh_uri.value.replace(prefix, ""));
}
```

#### `mesh_qnames` Joined MeSH ID QNames with the mesh: prefix

```javascript
({meshs}) => {
  return meshs.map(mesh => "mesh:" + mesh).join(" ");
}
```

## Endpoint

http://lsd.dbcls.jp/sparql

### `mesh2label` MeSH ID to LSD label

```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX lsd: <http://purl.jp/bio/10/lsd/ontology/201209#>
PREFIX mesh: <http://purl.jp/bio/10/lsd/mesh/>

SELECT ?mesh_uri (STR(?en) AS ?en_label)
WHERE {
   VALUES ?mesh_uri { {{mesh_qnames}} }
   ?mesh_uri rdfs:label ?en .
   FILTER(LANG(?en) = "en")
}
```

## `results` Compile results

```javascript
({ordered_pmids, pmid2reference, pmid2citation, mesh2label, pmid2mesh}) => {
  let articles = {};
  let mesh_lsd = {};

  ordered_pmids.forEach(pmid => {
    let pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    let pubtator = " (<a href=\"https://www.ncbi.nlm.nih.gov/CBBresearch/Lu/Demo/PubTator/curator_identifier.cgi?pmid=" + pmid + "&Gene_display=1&Disease_display=1&Mutation_display=1&Species_display=1&Chemical_display=1\">PubTator</a>)";
    articles[pmid] = {
      "pmid": pubmed + pubtator,
      "diseases": []
    };
  });

  pmid2reference.results.bindings.forEach(item => {
    let pmid = item.pmid.value;
    let html = "";
    html += "<b>" + item.title.value + "</b><br>\n";
    html += item.authors.value + "<br>\n";
    html += "<i><b>" + item.journal.value + "</b></i><br>\n";
    articles[pmid].reference = html;
    articles[pmid].year = item.year.value;
    // default value for citation
    articles[pmid].citation = "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + 0 + "</a>";
  });

  pmid2citation.results.bindings.forEach(item => {
    let pmid = item.pmid.value;
    articles[pmid].citation = "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + item.citation_count.value + "</a>";
  });

  mesh2label.results.bindings.forEach(item => {
    let mesh = item.mesh_uri.value.replace("http://purl.jp/bio/10/lsd/mesh/", "");
    mesh_lsd[mesh] = {};
    mesh_lsd[mesh].en = item.en_label.value || null;
  });

  pmid2mesh.results.bindings.forEach(item => {
    let pmid = item.pmid_uri.value.replace("http://identifiers.org/pubmed/", "");
    let mesh = item.mesh_uri.value.replace("http://identifiers.org/mesh/", "");
    let disease = "MeSH: " + "<a href=\"https://www.ncbi.nlm.nih.gov/mesh/?term=" + mesh + "\">" + mesh + "</a>";
    if (mesh_lsd[mesh]) {
      disease += " " + mesh_lsd[mesh].en;
    } else {
      disease += " (not found in LSD)";
    }
    articles[pmid].diseases.push(disease);
  });

  let results = {
    header: ["PMID", "Reference", "Year", "Cited by", "Diseases"],
    data: []
  };
  
  ordered_pmids.forEach(pmid => {
    let article = articles[pmid];
    results.data.push(
      [ article.pmid, article.reference, article.year, article.citation, article.diseases ]
    );
  });

  return results
}
```

## `result`

```javascript
({
  json({results}) {
    return {
      columns: results.header.map(x => [x]),
      data: results.data
    };
  }
})
```
