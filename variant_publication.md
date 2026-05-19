# TogoVar variant_publication stanza query

Generate rs2pubmed table data by dbSNP ID

## Parameters

* `rs` dbSNP ID
  * default: rs114202595
  * example: rs671(hit both), rs797044836(pubTatorCentral only), rs112750067(no hits)
* `snippet_source` source for snippets
  * default: ncbi
  * example: ncbi, pubannotation, hybrid, none

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

## `pmids_litvar` Obtain PMID by dbSNP ID using LitVar

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

## `pmids_pubtator_union_litvar` Concat PMIDs from Pubtator and LitVar

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

## `colil` PubMed IDs to their citation count

```javascript
async ({pmids_pubtator_union_litvar}) => {
  let pmids = pmids_pubtator_union_litvar.filter(pmid => pmid !== "no data");
  const PMIDS_PER_REQUEST = 300;

  let results = {};

  try {
    while(pmids.length > 0){
      let pmids_per_request =
        encodeURIComponent(pmids.splice(0, Math.min(PMIDS_PER_REQUEST, pmids.length)).map(num => String(num)).join(','));

      const sparqlist = ("/sparqlist/api/colil?pmids=").concat(pmids_per_request);
      const response = await fetch(sparqlist, {method: "get", headers: {"Accept": "application/json"}});
      if (!response.ok) {
        throw new Error("Failed to fetch from " + sparqlist);
      }

      const res = await response.json();
      results = { ...results, ...res };
    }
  } catch (error) {
    console.log(error);
    return null;
  }

  return results;
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

/**
 * Convert a list of PMIDs to PMCIDs using TogoID.
 * @param {string[]} pmids - e.g., ["12345", "31452104", "28924058"]
 * @param {number} chunkSize - optional chunk size for requests
 * @returns {Promise<Map<string, string[]>>} Map from PMID -> array of PMCIDs
 */
async function pmidsToPmcids(pmids, chunkSize = 300) {
  const base = "https://api.togoid.dbcls.jp/convert";
  const map = new Map();
  pmids.forEach(id => map.set(id, [])); // 初期化

  if (pmids.length === 0) {
    return map;
  }

  for (let i = 0; i < pmids.length; i += chunkSize) {
    const ids = pmids.slice(i, i + chunkSize);
    const params =
      "ids=" + encodeURIComponent(ids.join(",")) +
      "&route=" + encodeURIComponent("pubmed,pmc") +
      "&format=" + encodeURIComponent("json") +
      "&report=" + encodeURIComponent("pair");

    const res = await fetch(`${base}?${params}`);
    if (!res.ok) {
      throw new Error(`TogoID request failed: ${res.status} ${res.statusText}`);
    }

    const json = await res.json();
    const results = Array.isArray(json?.results) ? json.results : [];

    for (const pair of results) {
      const [pmid, pmcid] = Array.isArray(pair)
        ? pair
        : [pair?.source ?? pair?.from ?? pair?.pmid, pair?.target ?? pair?.to ?? pair?.pmcid];

      if (pmid && pmcid && map.has(String(pmid))) {
        map.get(String(pmid)).push(String(pmcid));
      }
    }
  }

  return map; // Map { PMID → [PMCID, ...] }
}

async function fetchPubAnnotationDocument(pmid, pmcids = []) {
  if (pmcids.length === 0) {
    return null;
  }

  const pmcUrls = pmcids.flatMap(pmcid => {
    return [
      "https://pubannotation.org/projects/PubTatorOnTogoVar/docs/sourcedb/PMC/sourceid/" + encodeURIComponent(String(pmcid)) + "/annotations.json"
    ];
  });

  const documents = await Promise.all(pmcUrls.map(async url => {
    try {
      const response = await fetch(url);
      if (!response.ok) {
        return null;
      }

      const json = await response.json();
      if (json?.text && Array.isArray(json?.denotations)) {
        return json;
      }
    } catch (error) {
      console.log(error);
    }

    return null;
  }));

  return documents.find(Boolean) || null;
}

async function fetchNcbiPubTatorDocuments(pmids, chunkSize = 100) {
  const docs = [];

  for (let i = 0; i < pmids.length; i += chunkSize) {
    try {
      const chunk = pmids.slice(i, i + chunkSize);
      const url = "https://www.ncbi.nlm.nih.gov/research/pubtator-api/publications/export/biocjson?pmids=" + encodeURIComponent(chunk.join(","));
      const response = await fetch(url);

      if (!response.ok) {
        throw new Error("Failed to fetch from " + url);
      }

      const json = await response.json();
      docs.push(...(Array.isArray(json?.PubTator3) ? json.PubTator3 : []));
    } catch (error) {
      console.log(error);
    }
  }

  return docs;
}

async function mapWithConcurrency(items, concurrency, mapper) {
  const results = [];
  let index = 0;

  const workers = Array.from({ length: Math.min(concurrency, items.length) }, async () => {
    while (index < items.length) {
      const currentIndex = index++;
      results[currentIndex] = await mapper(items[currentIndex], currentIndex);
    }
  });

  await Promise.all(workers);
  return results;
}

function annotationMatchesVariant(annotation, mutationId) {
  const infons = annotation?.infons || {};
  const rsids = Array.isArray(infons.rsids) ? infons.rsids.map(String) : [];
  const normalized = Array.isArray(infons.normalized) ? infons.normalized.map(String) : [];

  return (
    infons.rsid === mutationId ||
    rsids.includes(mutationId) ||
    infons.normalized_id === mutationId + "##" ||
    normalized.includes(mutationId + "##")
  );
}

function buildSnippetFromNcbiPubTatorDocument(doc, mutationId, snippetMaxLength) {
  const snippets = [];

  for (const passage of doc.passages || []) {
    const text = passage?.text || "";
    const passageOffset = Number(passage?.offset || 0);
    const annotations = Array.isArray(passage?.annotations) ? passage.annotations : [];

    const markup_ranges = annotations
      .filter(annotation => annotationMatchesVariant(annotation, mutationId))
      .flatMap(annotation => Array.isArray(annotation?.locations) ? annotation.locations : [])
      .map(location => {
        const begin = Number(location.offset) - passageOffset;
        const end = begin + Number(location.length);
        return { begin, end };
      })
      .filter(({ begin, end }) => begin >= 0 && end > begin && end <= text.length);

    if (markup_ranges.length === 0) {
      continue;
    }

    const snippet = create_snippet(text, markup_ranges, snippetMaxLength);
    if (snippet) {
      snippets.push(snippet);
    }
  }

  return snippets.join(" ... ");
}

function buildSnippetFromPubAnnotationDocument(jsonInput, mutationId, snippetMaxLength) {
  const text = jsonInput?.text || "";
  const denotations = Array.isArray(jsonInput?.denotations) ? jsonInput.denotations : [];
  const attributes = Array.isArray(jsonInput?.attributes) ? jsonInput.attributes : [];
  const rsNumber = mutationId.replace(/^rs/i, "");

  const denotations_incl_mutationId = attributes.filter(attr => {
    const obj = String(attr?.obj || "");
    return (
      obj === mutationId ||
      obj.includes(mutationId + "##") ||
      obj.includes("tmVar:" + mutationId + ";") ||
      obj.includes("RS#:" + rsNumber + ";") ||
      obj.endsWith("RS#:" + rsNumber)
    );
  });
  const markup_ranges = denotations_incl_mutationId
    .map(denotation_incl_mutationId => denotations.find(d => d.id === denotation_incl_mutationId.subj))
    .filter(Boolean)
    .map(denotation => denotation.span)
    .filter(span => span && Number.isInteger(span.begin) && Number.isInteger(span.end))
    .map(span => ({ begin: span.begin, end: span.end }));

  if (markup_ranges.length === 0) {
    return "";
  }

  return create_snippet(text, markup_ranges, snippetMaxLength);
}



async({rs, pubtator, snippet_source}) => {
  const mutationId = rs;
  const snippet_maxlength = 300;
  const snippets = {};
  const source = (snippet_source || "ncbi").toLowerCase();

  if (source === "none" || source === "off" || source === "false") {
    return snippets;
  }

  const pmids = Object.keys(pubtator);

  if (source === "ncbi" || source === "pubtator" || source === "hybrid") {
    try {
      const docs = await fetchNcbiPubTatorDocuments(pmids);
      for (const doc of docs) {
        const pmid = String(doc?.id || doc?.pmid || "");
        const snippet = buildSnippetFromNcbiPubTatorDocument(doc, mutationId, snippet_maxlength);
        if (pmid && snippet) {
          snippets[pmid] = snippet;
        }
      }
    } catch (error) {
      console.log(error);
    }

    if (source !== "hybrid") {
      return snippets;
    }
  }

  if (source !== "pubannotation" && source !== "pubannotator" && source !== "hybrid") {
    return snippets;
  }

  const pmidToPmcids = await pmidsToPmcids(pmids);
  const pubannotationPmids = pmids.filter(pmid => !(pmid in snippets));

  await mapWithConcurrency(pubannotationPmids, 8, async pmid => {
    try {
      const jsonInput = await fetchPubAnnotationDocument(pmid, pmidToPmcids.get(pmid) || []);
      if (jsonInput === null) {
        return;
      }

      const snippet = buildSnippetFromPubAnnotationDocument(jsonInput, mutationId, snippet_maxlength);
      if (snippet) {
        snippets[pmid] = snippet;
      }
    } catch (error) {
      console.log(error);
    }
  });

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
    const href_litvar = "<br>(<a href=\"https://www.ncbi.nlm.nih.gov/research/litvar2/publication/" + pmid + "?variant=litvar%40" + rs + "%23%23\">LitVar2</a>)";

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
      html += "<br>\n";
      html += "<span style=\"display: inline-block; padding: 0.2rem 0.45rem; border-left: 3px solid #777; background-color: #f5f5f5; color: #222;\">Mentioned text: " + snippets[pmid] + "</span><br>\n";
    }

    articles.push([
      links,
      html,
      pubtator_litvar[pmid].year.split(" ")[0],
      "<a href=\"http://colil.dbcls.jp/browse/papers/" + pmid + "/\">" + (colil === null ? "N/A" : (colil[pmid] == undefined ? 0 : colil[pmid])) + "</a>"
    ]);
  }

  return {
    columns: [["PMID"], ["Reference"], ["Year"], ["Cited by"]],
    data: articles
  };
}
```
