# clinvar_mondo_test

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `clinvar` ClinVarID
  * default: 88996

## Endpoint

{{ ep }}

## `clinvar2medgen`

```sparql

PREFIX clinvar: <http://ncbi.nlm.nih.gov/clinvar/variation/>
PREFIX m2r: <http://med2rdf.org/ontology/med2rdf#>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX medgen: <http://ncbi.nlm.nih.gov/medgen/>

select distinct ?clinvar_uri ?medgen_uri
where {
  GRAPH <http://togovar.biosciencedbc.jp/clinvar>{
    clinvar:{{clinvar}} m2r:disease ?_o1.
    ?_o1  dcterms:references ?_o2 .
    ?_o2 rdfs:seeAlso ?medgen_uri .
    ?clinvar_uri m2r:disease ?_o1.
    
    #FILTER REGEX (?clinvar_uri , clinvar: ).
    FILTER REGEX (?medgen_uri , medgen: ).
  }
}
```

## `medgenid`

```javascript
({clinvar2medgen}) => {
  let prefix = "http://ncbi.nlm.nih.gov/medgen/";
  
  return clinvar2medgen.results.bindings.map(x => x.medgen_uri.value.replace(prefix, ""));
}
```

## `medgen2omim`

```sparql

PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX medgen: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX omim: <http://purl.bioontology.org/ontology/OMIM/>
#URI統一後は下のURIを用いる
#PREFIX medgen: <http://ncbi.nlm.nih.gov/medgen/>
#PREFIX omim: <http://identifiers.org/omim/>

SELECT DISTINCT ?omim_uri ?medgen_condition
WHERE {
GRAPH <http://togovar.biosciencedbc.jp/medgen>{
    medgen:{{medgenid}} rdfs:label ?medgen_condition ;
      rdfs:seeAlso ?omim_uri.
  
    FILTER REGEX (?omim_uri , omim: ).
 } 
}
```

## `omimid`

```javascript
({medgen2omim}) => {
  let prefix = "http://purl.bioontology.org/ontology/OMIM/";
  
  return medgen2omim.results.bindings.map(x => x.omim_uri.value.replace(prefix, ""));
}
```

## `omim2mondo`

```sparql

PREFIX core: <http://www.w3.org/2004/02/skos/core#>
PREFIX omim: <http://identifiers.org/omim/>

SELECT DISTINCT ?mondo_uri
WHERE {
 GRAPH <http://togovar.biosciencedbc.jp/mondo>{
       ?mondo_uri core:exactMatch omim:{{omimid}} .
 }  
}
```

## `mondoid`

```javascript
({omim2mondo}) => {
  let prefix = "http://purl.obolibrary.org/obo/MONDO_";
  
  return omim2mondo.results.bindings.map(x => x.mondo_uri.value.replace(prefix, ""));
}
```

## `mondo2trait`
```sparql

PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX gwas_dataset: <http://rdf.ebi.ac.uk/dataset/gwas/>
PREFIX gwas_terms: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX mondo: <http://purl.obolibrary.org/obo/MONDO_>

SELECT DISTINCT ?efo ?gwas ?p4 ?o3
FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
FROM <http://togovar.biosciencedbc.jp/efo>
WHERE {  


#?s ?p mondo:{{mondoid}};
?efo ?p "OMIM:{{omimid}}"^^<http://www.w3.org/2001/XMLSchema#string>;
  ?p2 ?o2.

 FILTER (REGEX (?efo , "http://www.ebi.ac.uk/efo/" )).
# FILTER (REGEX (?p2 , "http://www.ebi.ac.uk/" )).
# FILTER (REGEX (?o , "OMIM:" )).
# FILTER (REGEX (?p2 , "http://rdf.ebi.ac.uk/terms/gwas/" )).
# FILTER (REGEX (?p2 , "http://www.w3.org/2000/01/rdf-schema#label" )).

?gwas ?p3 ?efo;
  ?p4 ?o3.

 FILTER (REGEX (?p4 , "http://rdf.ebi.ac.uk/terms/gwas/has_gwas_trait_name" )).
}

```

## `results`

```javascript
({
  json({clinvar2medgen, medgen2omim, omim2mondo}) {
    let clinvar_id = "";
    let medgen_condition = "";
    let mondo_uri = "";
    
    clinvar_id = clinvar2medgen.results.bindings.map(x => x.clinvar_uri.value);
    medgen_condition = medgen2omim.results.bindings.map(x => x.medgen_condition.value);
    mondo_uri = omim2mondo.results.bindings.map(x => x.mondo_uri.value);
  
    return {
      data: [
        {"ClinVar ID": clinvar_id, "MedGen Condition": medgen_condition, "MONDO URI": mondo_uri}
      ]
    };
  }
})
```