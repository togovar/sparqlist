# Disease report / Summary

## Parameters

* `medgen_cid` MedGen CID
  * default:
  * example: C0023467, C2675520, C2608086

## Endpoint

{{SPARQLIST_TOGOVAR_SPARQL}}

## `medgen`

```sparql
PREFIX dct:      <http://purl.org/dc/terms/>
PREFIX medgen:   <http://www.ncbi.nlm.nih.gov/medgen/>
PREFIX mo:       <http://med2rdf/ontology/medgen#>
PREFIX oboinowl: <http://www.geneontology.org/formats/oboInOwl#>
PREFIX rdfs:     <http://www.w3.org/2000/01/rdf-schema#>
PREFIX skos:     <http://www.w3.org/2004/02/skos/core#>

SELECT DISTINCT ?medgen_cid ?medgen_label ?medgen_definition ?mondo ?efo ?mesh
WHERE {
  VALUES ?medgen { medgen:{{medgen_cid}} }

  GRAPH <http://togovar.org/medgen> {
    ?medgen a mo:ConceptID;
      dct:identifier ?medgen_cid;
      rdfs:label ?medgen_label .
    OPTIONAL {
      ?medgen skos:definition ?medgen_definition .
    }

    OPTIONAL {
      ?medgen mo:mgconso ?mgconso_mondo.
      ?mgconso_mondo dct:source mo:MONDO ;
        rdfs:seeAlso ?mondo .

      OPTIONAL {
        GRAPH <http://togovar.org/mondo> {
          ?mondo oboinowl:hasDbXref ?dbxref_efo .
          FILTER STRSTARTS(STR(?dbxref_efo), "EFO:")
          BIND(IF(STRLEN(?dbxref_efo) > 0, URI(CONCAT("http://www.ebi.ac.uk/efo/EFO_", SUBSTR(?dbxref_efo,5))), URI("")) AS ?efo)
        }
      }

      OPTIONAL {
        GRAPH <http://togovar.org/mondo> {
          ?mondo oboinowl:hasDbXref ?dbxref_mesh.
          FILTER STRSTARTS(STR(?dbxref_mesh), "MESH:")
          BIND(IF(STRLEN(?dbxref_mesh) > 0, URI(CONCAT("http://id.nlm.nih.gov/mesh/", SUBSTR(?dbxref_mesh,6))), URI("")) AS ?mesh)
        }
      }
    }
  }
}
```

## `result`

```javascript
({medgen}) => {
  const d = medgen.results.bindings[0];
  if(d == null){
    return [];
  }

  const medgen_definition = d.medgen_definition ? d.medgen_definition.value : ""
  const efo_link = d.efo ? `EFO:&ensp;<a href="${d.efo.value}">${d.efo.value.replace("http://www.ebi.ac.uk/efo/", "")}</a>` : "EFO:&ensp;No Data";
  const medgen_link = d.medgen_cid ? `MedGen:&ensp;<a href="https://www.ncbi.nlm.nih.gov/medgen/${d.medgen_cid.value}">${d.medgen_cid.value}</a>` : ""
  const mesh_link = d.mesh ? `MeSH:&ensp;<a href="${d.mesh.value}">${d.mesh.value.replace("http://id.nlm.nih.gov/mesh/", "")}</a>` : "MeSH:&ensp;No Data"
  const mondo_link = d.mondo ? `MONDO:&ensp;<a href="${d.mondo.value.replace("http://purl.obolibrary.org/obo/MONDO_", "https://monarchinitiative.org/disease/MONDO:")}">${d.mondo.value.replace("http://purl.obolibrary.org/obo/MONDO_", "MONDO:")}</a>` : "MONDO:&ensp;No Data";

  return [{
    label: d.medgen_label.value,
    definition: medgen_definition,
    links: [efo_link, medgen_link, mesh_link, mondo_link].join("&emsp;"),
  }];
}
```
