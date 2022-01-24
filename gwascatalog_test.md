# gwas_test

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `clinvar` Clinvar ID
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



## `getgwas`
```sparql

PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX gwas_dataset: <http://rdf.ebi.ac.uk/dataset/gwas/>
PREFIX gwas_terms: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX mondo: <http://purl.obolibrary.org/obo/MONDO_>
PREFIX efo: <http://www.ebi.ac.uk/efo/EFO_>


SELECT DISTINCT ?trackable ?p_value ?trait_name ?efo_uri ?rs
FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
FROM <http://togovar.biosciencedbc.jp/efo>
FROM <http://togovar.biosciencedbc.jp/medgen>
FROM <http://togovar.biosciencedbc.jp/mondo>
WHERE {  

# VALUES ?s { <http://rdf.ebi.ac.uk/dataset/gwas/Trackable/16617> }.
# VALUES ?s { <http://rdf.ebi.ac.uk/dataset/gwas/Trackable/16036445> }.

?efo_uri ?p "OMIM:{{omimid}}"^^<http://www.w3.org/2001/XMLSchema#string>.
  
?trackable <http://purl.org/oban/has_object> ?efo_uri;
   <http://rdf.ebi.ac.uk/terms/gwas/has_p_value> ?p_value;
   <http://rdf.ebi.ac.uk/terms/gwas/has_gwas_trait_name> ?trait_name;
   <http://www.w3.org/2000/01/rdf-schema#label> ?rs.
#?s ?p2 ?o2.
  
#?efo_uri ?p ?o.
  
  
# from mondo_test
#?efo ?p "OMIM:{{omimid}}"^^<http://www.w3.org/2001/XMLSchema#string>;
#  ?p2 ?o2.

# FILTER (REGEX (?efo , "http://www.ebi.ac.uk/efo/" )).
#?gwas ?p3 ?efo;
#  ?p4 ?o3.
# FILTER (REGEX (?p4 , "http://rdf.ebi.ac.uk/terms/gwas/has_gwas_trait_name" )).

}
```

## `result`

```javascript
({
  json({getgwas}) {
    let articles = {};
    let p_value = {};
    let trait_name = {};
    let efo_uri = {};
    let rs_uri = {};
    
    let trackables = [];
    let trackable = "";
    let rs = "";
    let rs_start_index ="";
    let rs_end_index ="";

    if (getgwas.results.bindings) {    
      getgwas.results.bindings.forEach(x => {
        trackable = x.trackable.value;
        trackables.push(trackable);
        
        p_value[trackable] = x.p_value.value;
        trait_name[trackable] = x.trait_name.value;
        efo_uri[trackable] = x.efo_uri.value;
        rs = x.rs.value + "";
        rs_start_index = rs.indexOf(' rs') + 1;
        rs_end_index = rs.indexOf(' ', rs_start_index) - 1;
        rs_uri[trackable] = "https://www.ebi.ac.uk/gwas/variants/" + rs.substr(rs_start_index, rs_end_index - rs_start_index);
      });

    }
    return {      
      columns: [["P Value"], ["Trait name"], ["EFO URI"], ["rs URI"]],
      data: trackables.map((x, index) =>  {
        return [p_value[x], trait_name[x], efo_uri[x], rs_uri[x]]
      })
  };
  }
})
```