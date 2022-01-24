# Disease report / Base

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `disease` disease name
  * default: Breast-ovarian cancer, familial 2

## Endpoint

{{{ep}}}

## `getSynonyms`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX medgen: <http://med2rdf/ontology/medgen#>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>

SELECT DISTINCT ?medgen_uri ?synonyms

WHERE {

  ?medgen_uri rdfs:label "{{disease}}" ;
    rdfs:seeAlso ?medgen_cid_uri.

  ?medgen_cid_uri rdf:type medgen:ConceptID ;
    medgen:mgconso ?synonym_node.
    FILTER ( strstarts(str(?synonym_node), "nodeID://") )
  
  ?synonym_node rdfs:label ?synonyms.
    FILTER ( !strstarts(str(?synonyms), "{{disease}}") )

  }

```

## `medgenInfo`

```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX medgen: <http://med2rdf/ontology/medgen#>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>


SELECT DISTINCT ?medgen_uri ?concept_id ?gene_location ?gene_name ?omim ?definition

WHERE {

  ?medgen_uri rdfs:label "{{disease}}" ;
    rdfs:seeAlso ?medgen_cid_uri.

  ?medgen_cid_uri skos:definition ?definition ;
    rdf:type medgen:ConceptID ;
    dct:identifier ?concept_id.

  ?medgen_cid_uri medgen:mgsat ?gene_location_node ;
    medgen:mgsat ?gene_name_node ;
    medgen:mgsat ?omim_node .
  
  ?gene_location_node rdfs:label "NCBI_CYTOGEN_LOC" ;
    rdf:value ?gene_location.

  ?gene_name_node rdfs:label "GENESYMBOL" ;
    rdf:value ?gene_name.

  ?omim_node rdfs:label "NCBI_OMIM";
    rdf:value ?omim.

  
}
```

## `result`

```javascript
({
  json({getSynonyms, medgenInfo}){
    let medgen_id = "";
    let synonyms = "";
    let concept_id ="";
    let gene_location = "";
    let gene_names = "";
    let omim = "";
    let definition= "";
    
    
    getSynonyms.results.bindings.forEach(x => {
      if (synonyms == "" ){
        medgen_id = x.medgen_uri.value.split('/').slice(-1)[0];
        synonyms = x.synonyms.value;      
      } else {
        synonyms += ", " +x.synonyms.value; 
      }
    });
    
    medgenInfo.results.bindings.forEach(x => {
      if (gene_names == ""){
        concept_id = x.concept_id.value;
        gene_location = x.gene_location.value;
        gene_names = x.gene_name.value;
        omim = x.omim.value;
        definition = x.definition.value;
      } else {
        gene_names += ", " + x.gene_name.value;
      }
    });
    
    return {
      data: [
        {"MedGenid": medgen_id, "Genes": gene_names + "(" + gene_location + ")", "Synonyms": synonyms, "OMIM": omim, "HPO": "", "Description": definition}
      ]
    };
  }
})
```