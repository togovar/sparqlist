# Disease name expander for Advanced search
* A disease name in terms of MedGen is expanded to subtypes by using the Mesh tree  

# Parameters
* `disease_name_in_medgen` (Exact MedGen term including case-sensitivity must be specified) 
  * default: 
  * example: Leukemia (https://www.ncbi.nlm.nih.gov/medgen/9725), Acute myeloid leukemia (https://www.ncbi.nlm.nih.gov/medgen/9730)   

# Output
* medgen_in: URI of disease_name_in_medgen 
* medgen_in_label: disease_name_in_medgen
* medgen_subtype: URI of the subtypes of the medgen_in
* medgen_subtype_label: Label of medgen_subtype

  
# TestURL
- [disease_name_in_medgen=Leukemia](https://togovar-dev.biosciencedbc.jp/sparqlist/api/advanced_search_disease_name_expander?disease_name_in_medgen=Leukemia)
- [disease_name_in_medgen="Acute myeloid leukemia"](https://togovar-dev.biosciencedbc.jp/sparqlist/api/advanced_search_disease_name_expander?disease_name_in_medgen=Acute%20myeloid%20leukemia)

## Endpoint
https://togovar-dev.biosciencedbc.jp/sparql

## `data`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX mesh: <http://id.nlm.nih.gov/mesh/>
PREFIX meshv: <http://id.nlm.nih.gov/mesh/vocab#>
PREFIX dct: <http://purl.org/dc/terms/>
PREFIX medgen_ncbi: <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX medgen_med2rdf: <http://med2rdf/ontology/medgen#>
PREFIX cvo:  <http://purl.jp/bio/10/clinvar/>

SELECT DISTINCT ?medgen_in ?medgen_in_label ?medgen_subtype ?medgen_subtype_label 
FROM <http://togovar.biosciencedbc.jp/mesh>
FROM <http://togovar.biosciencedbc.jp/medgen>
WHERE {
    VALUES ?medgen_in_label { "{{ disease_name_in_medgen }}" }
  
  # input medgen_id (C0023467) to mesh_descripotor (D015470)
   GRAPH <http://togovar.biosciencedbc.jp/medgen>{
     ?medgen_in rdfs:label ?medgen_in_label.
     ?medgen_in medgen_med2rdf:mgconso ?mgconso_in.
     ?mgconso_in dct:source medgen_med2rdf:MSH. 
     ?mgconso_in rdfs:seeAlso ?mesh_in.
   }
   
  # mesh descripotor (D015470) to mesh subtypecategories (D004915, D007946)
   GRAPH <http://togovar.biosciencedbc.jp/mesh>{
     ?mesh_in meshv:treeNumber ?mesh_in_tree_number .
     ?mesh_in rdfs:label ?mesh_in_label .
     ?mesh_subtype meshv:treeNumber/meshv:parentTreeNumber* ?mesh_in_tree_number .
     ?mesh_subtype rdfs:label ?mesh_subtype_label.
     ?mesh_subtype meshv:treeNumber ?mesh_subtype_tree_number.
  }
  
  # mesh subtypecategories (D004915, D007946...) to medgen  (C0023440, C0023461...)
   GRAPH <http://togovar.biosciencedbc.jp/medgen>{
     ?mgconso_subtype rdfs:seeAlso ?mesh_subtype.
     ?mgconso_subtype dct:source medgen_med2rdf:MSH. 
     ?medgen_subtype medgen_med2rdf:mgconso ?mgconso_subtype.
     ?medgen_subtype rdfs:label ?medgen_subtype_label.  
  }
  
} 
```

## `return`
- 整形
```javascript
({data})=>{
  return data.results.bindings.map(d=>{ 
    return {
      medgen_in: d.medgen_in.value, 
      medgen_in_label: d.medgen_in_label.value, 
      medgen_subtype: d.medgen_subtype.value, 
      medgen_subtype_label: d.medgen_subtype_label.value, 
    };
  });	
}
```
