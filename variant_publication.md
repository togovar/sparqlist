# TogoVar variant_publication stanza query

Generate rs2pubmed table data by dbSNP ID

## Parameters

* `rs` dbSNP ID
  * default: rs114202595
  * example: rs671(hit both), rs797044836(pubTatorCentral only), rs112750067(no hits)

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `rs2pubtator` dbSNP ID to PubMed Info by Pubtator and PubMed

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

## `pubtator` Reformat JSON from PubTator

```javascript
({rs2pubtator}) => {
  const ref = {};

  rs2pubtator.results.bindings.forEach(x => {
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

## `pmids_litvar` Obtain PMID by dbSNP ID using Litvar

```javascript
async ({rs}) => {
  try {
    const res = await fetch("https://www.ncbi.nlm.nih.gov/research/litvar2-api/variant/get/litvar%40" + rs + "%23%23/publications?format=json", {
      method: 'GET',
      headers: {
        'Accept': 'application/json',
        'Content-Type': 'application/x-www-form-urlencoded'
      }
    }).then(res => res.json());

    return res.pmids.map(pmid => pmid.toString()) ;
  } catch (error) {
    console.log(error);
    return []
  }
};
```

## `pmids_litvar_minus_pubtator` Remove PMIDs found in PubTator from LitVar

```javascript
({pmids_litvar, pubtator}) => {
  const ret = pmids_litvar.filter(pmid => Object.keys(pubtator).indexOf(pmid) == -1)

  return ret
}
```

## `pmids_pubtator_union_litvar` Concat PMIDs from Pubtator and Litvar

```javascript
({pmids_litvar_minus_pubtator, pubtator}) => {
  const pmids_pubtator = Object.keys(pubtator)
  const pmids_pubtator_union_litvar = pmids_pubtator.concat(pmids_litvar_minus_pubtator)

  if(pmids_pubtator_union_litvar.length == 0){
    pmids_pubtator_union_litvar.push("no data")
  }

  return pmids_pubtator_union_litvar
}
```

## `pubmed_litvar_only` Get bibliography from PubMed for PMIDs included in LitVar only

```sparql
PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX dct:    <http://purl.org/dc/terms/>
PREFIX olo:    <http://purl.org/ontology/olo/core#>
PREFIX foaf:   <http://xmlns.com/foaf/0.1/>
PREFIX pubmed: <http://rdf.ncbi.nlm.nih.gov/pubmed/>

SELECT DISTINCT ?pmid_uri ?pmid ?title ?year ?author ?journal
WHERE {
  VALUES ?pmid_uri { {{#each pmids_litvar_minus_pubtator}}pubmed:{{this}} {{/each}} }

  GRAPH <http://togovar.org/pubmed> {
    ?pmid_uri bibo:pmid ?pmid ;
      dct:title ?title ;
      dct:issued ?year ;
      dct:creator/olo:slot/olo:item/foaf:name ?author ;
      dct:source ?journal .
  }
}
```

## `litvar_only` Reformat JSON from LitVar

```javascript
({pubmed_litvar_only}) => {
  const ref = {}

  pubmed_litvar_only.results.bindings.forEach(x => {
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

## Endpoint

https://colil.dbcls.jp/sparql

## `colil_sparql` Get citation count from Colil

```sparql
DEFINE sql:select-option "order"

PREFIX bibo:   <http://purl.org/ontology/bibo/>
PREFIX colil:  <http://purl.jp/bio/10/colil/ontology/201303#>
PREFIX togows: <http://togows.dbcls.jp/ontology/ncbi-pubmed#>
PREFIX rdfs:   <http://www.w3.org/2000/01/rdf-schema#>

SELECT ?pmid (COUNT(?citation_paper) AS ?citation_count)
WHERE {
  VALUES ?pmid { {{#each pmids_pubtator_union_litvar}}'{{this}}' {{/each}} }

  GRAPH <http://purl.jp/bio/10/colil/core> {
    ?pmid ^togows:pmid ?pubmed .
    ?pubmed a colil:PubMed ;
      ^rdfs:seeAlso/^bibo:cites ?citation_paper.
  }
}
```

## `colil` Reformat JSON from Colil

```javascript
({colil_sparql}) => {
  const colil = {}

  colil_sparql.results.bindings.forEach(x => {
    const pmid = x.pmid.value;
    colil[pmid] = x.citation_count.value;
    }
  );

  return colil
}
```

## `snippets` Generate snippets

```javascript
function create_snippet(text, markup_ranges, max_length) {
  // Regular expression to split sentences, considering abbreviations
  const sentence_regex = /(?<!\b(?:Mr|Mrs|Ms|Dr|Prof|Sr|Jr)\.)(?<!\b\w\.\w\.\w\.)(?<![A-Z]\.)(?<!\b[a-z]\.)(?<!\betc\.)(?<!\b[A-Z]{2}\.)(?<=[.!?])\s+(?=[A-Z])/g;

  // Apply <b></b> tags to the specified markup ranges
  let marked_up_text = "";
  let last_index = 0;

  markup_ranges.forEach(({begin, end}) => {
    marked_up_text += text.slice(last_index, begin) + "<b>" + text.slice(begin, end) + "</b>";
    last_index = end;
  });

  marked_up_text += text.slice(last_index);

  // Split the text into sentences
  const sentences = marked_up_text.split(sentence_regex);

  // Generate the snippet
  let snippet = "";
  let length = 0;
  let is_first_sentence = true;
  let previous_sentence_had_markup = false;

  for (let i = 0; i < sentences.length; i++) {
    const sentence = sentences[i];

    // Skip sentences that don't contain any markup
    if (!sentence.includes('<b>') && !sentence.includes('</b>')) {
      previous_sentence_had_markup = false;
      continue;
    }

    // Check if adding this sentence exceeds max_length
    if (length + sentence.length > max_length && !is_first_sentence) {
      // Check if there are any remaining sentences with markup after this one
      if (sentences.slice(i).some(s => s.includes('<b>'))) {
        snippet += " ...";
      }
      break;
    }

    // If not the first sentence and the previous sentence did not have markup, add " ... "
    if (!is_first_sentence && !previous_sentence_had_markup) {
      snippet += " ... ";
    }

    snippet += sentence;
    length += sentence.length;
    is_first_sentence = false;
    previous_sentence_had_markup = true;
  }

  return snippet.trim();
}

async({rs, pubtator}) => {
  const mutationId = rs;
  const snippet_maxlength = 300;
  const snippets = {};

  for (const pmid in pubtator) {
    const url = "https://pubannotation.org/projects/PubTator4TogoVar/docs/sourcedb/PubMed/sourceid/" + pmid + "/annotations.json";

    try {
      const response = await fetch(url);
      if (!response.ok) { throw new Error("Failed to fetch from " + url); }

      const jsonInput = await response.json();
      const text = jsonInput.text;
      const denotations = jsonInput.denotations;

      // Find the denotation object for the mutationId
      const denotations_incl_mutationId = jsonInput.attributes.filter(attr => attr.obj === mutationId);

      // Get array of ranges for markup
      const markup_ranges = denotations_incl_mutationId.map(denotation_incl_mutationId => {
        return { begin, end } = denotations.find(d => d.id === denotation_incl_mutationId.subj).span;
      });

      // create snippet
      snippet = create_snippet(text, markup_ranges, snippet_maxlength);
      snippets[pmid] = snippet;
    } catch (error) {
      console.log(error);
    }
  }

  return snippets;
}
```

## `result` Compile results

```javascript
({rs, pmids_litvar, pubtator, litvar_only, colil, snippets}) => {
  const articles = [];
  const pmids_pubtator = Object.keys(pubtator);
  const pubtator_litvar = Object.assign(pubtator, litvar_only);

  for (const pmid in pubtator_litvar) {
    const href_pubmed = "<a href=\"https://www.ncbi.nlm.nih.gov/pubmed/" + pmid + "\">" + pmid + "</a>";
    const href_pubtator = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/pubtator3/publication/" + pmid + "\">PubTator3</a>)";
    const href_litvar = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/litvar2/publication/" + pmid + "?variant=litvar%40" + rs + "%23%23\">Litvar2</a>)";

    let links = href_pubmed;
    if (pmid in pubtator) {
      links += href_pubtator;
    }
    if (pmids_litvar.includes(pmid)) {
      links += href_litvar;
    }

    let html = "";
    html += "<b>" + pubtator_litvar[pmid].title + "</b><br>\n";
    html += pubtator_litvar[pmid].author + "<br>\n";
    html += "<i><b>" + pubtator_litvar[pmid].journal + "</b></i><br>\n";
    if (pmid in snippets) {
      html += "</br>" + snippets[pmid] + "</br>\n";
    }

    articles.push([
      links,
      html,
      pubtator_litvar[pmid].year.split(" ")[0],
      "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + (colil[pmid] == undefined ? 0 : colil[pmid]) + "</a>"
    ]);
  }

  return {
    columns: [["PMID"], ["Reference"], ["Year"], ["Cited by"]],
    data: articles
  };
}
```
