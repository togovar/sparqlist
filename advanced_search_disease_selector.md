# Disease selector for Advanced search

# Parameters

* `mesh_descriptor` (type: mesh descriptor)
  * default: 
  * example: なし(MeSH TreeのRoot(Diseases[C])),D007938(Childにmedgenない例), D007942(medgenがない例), D007951 (medgenが存在する例)

## `top`
- Top レベルかどうかのチェック
```javascript
({mesh_descriptor})=>{
  if (mesh_descriptor) return false;
  return true;
}
```

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
PREFIX sty: <http://purl.bioontology.org/ontology/STY/>

SELECT ?mesh_child_tree_number ?mesh_child ?mesh_child_label (GROUP_CONCAT(DISTINCT ?medgen_child_label, "|") AS ?medgen_child_labels) count(distinct ?mesh_offsprings) -1 AS ?mesh_offsprings_count
FROM <http://togovar.biosciencedbc.jp/mesh>
FROM <http://togovar.biosciencedbc.jp/medgen>
WHERE {
{{#if top}}
  # MeSH TreeのRoot(Diseases[C]) のURI もラベルもないので、その下の階層(Infections[C01],...)のDescriptor(D007239)を列挙する
  # mesh:D000820 (Animal diseases [C22])を除く 
  # See https://meshb.nlm.nih.gov/treeView
  VALUES ?mesh_child { mesh:D007239 mesh:D009369 mesh:D009140 mesh:D004066 mesh:D009057 mesh:D012140 mesh:D010038 mesh:D009422 mesh:D005128 mesh:D052801 mesh:D005261 mesh:D002318 mesh:D006425 mesh:D009358 mesh:D017437 mesh:D009750 mesh:D004700 mesh:D0071154 mesh:D007280 mesh:D013568 mesh:D009784 }
{{else}}
  # Top レベルでないノードは?mesh_inの直下?mesh_childを求める
  VALUES ?mesh_in {  <http://id.nlm.nih.gov/mesh/{{mesh_descriptor}}> }

  # input mesh descriptor (D015470) to mesh children (D004915, D007946)
  GRAPH <http://togovar.biosciencedbc.jp/mesh>{
    ?mesh_in meshv:treeNumber ?mesh_in_tree_number .
    ?mesh_in rdfs:label ?mesh_in_label .
    ?mesh_child meshv:treeNumber/meshv:parentTreeNumber ?mesh_in_tree_number .
  }
{{/if}}

  GRAPH <http://togovar.biosciencedbc.jp/mesh>{
    ?mesh_child rdfs:label ?mesh_child_label.
    ?mesh_child meshv:treeNumber ?mesh_child_tree_number.
    FILTER NOT EXISTS{
      ?mesh_child meshv:treeNumber ?mesh_child_tree_number2.
      FILTER(STRSTARTS(STR(?mesh_child_tree_number2), "http://id.nlm.nih.gov/mesh/C22")).
    }
  }

  # mesh childeren (D004915, D007946...) to medgen (C0023440, C0023461...)
  # medgen must be classified in "Disease or Syndrome" of Semantics Types Ontology (https://bioportal.bioontology.org/ontologies/STY?p=classes&conceptid=T047) 
  GRAPH <http://togovar.biosciencedbc.jp/medgen>{
    OPTIONAL {
     ?mgconso_child rdfs:seeAlso ?mesh_child.
     ?mgconso_child dct:source medgen_med2rdf:MSH.
     ?medgen_child medgen_med2rdf:mgconso ?mgconso_child.
     ?medgen_child rdfs:label ?medgen_child_label.
    }
  }

  # mesh children (D015470) to mesh offsprings (D004915, D007946)
  GRAPH <http://togovar.biosciencedbc.jp/mesh>{
    OPTIONAL {
      ?mesh_offsprings meshv:treeNumber/meshv:parentTreeNumber ?mesh_child_tree_number
    }
  }

}
GROUP BY ?mesh_child_tree_number ?mesh_child ?mesh_child_label
ORDER BY ?mesh_child_tree_number
```

## `return`
- 整形
```javascript
({data})=>{
  const idPrefix = "http://id.nlm.nih.gov/mesh/";
  let results = [];

  data.results.bindings.forEach(d=>{
    const medgen_labels = d.medgen_child_labels.value;
    const mesh_label = d.mesh_child_label.value;
    const has_child = (Number(d.mesh_offsprings_count.value) > 0 ? true : false); 

    if(medgen_labels){
      results = results.concat(medgen_labels.split('|').map(medgen_label=>{
        return {
          mesh_descriptor: d.mesh_child.value.replace(idPrefix,''),
          label : medgen_label,
          hasChild : has_child
        }
      }));
    }
  });

 return results;
}
```
