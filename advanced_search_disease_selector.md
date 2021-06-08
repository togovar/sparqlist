# Disease selector (Mesh Diseases category [C] 階層から))

# Parameters

* `mesh_in` (type: mesh descriptor) 
  * default: 
  * example: なし ([MeSH Diseases[C]](https://meshb.nlm.nih.gov/treeView)), D007938(Childにmedgenない例),D007942(medgenがない例), D007951 (medgenが存在する例)
  
## `top`
- Top レベルかどうかのチェック
```javascript
({mesh_in})=>{
  if (mesh_in) return false;
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
PREFIX cvo: <http://purl.jp/bio/10/clinvar/>

SELECT DISTINCT
  ?mesh_child ?mesh_child_label (SAMPLE(?mesh_child_tree_number) AS ?mesh_child_tree) 
  (GROUP_CONCAT(DISTINCT ?medgen_child_label, "|") AS ?medgen_child_labels)
  count(distinct ?mesh_offsprings) -1 AS ?mesh_offsprings_count
#  ?medgen_child ?medgen_child_label
#  count(distinct ?medgen_offsprings) -1 AS ?medgen_offsprings_count
#  ?medgen_offsprings_label 
FROM <http://togovar.biosciencedbc.jp/mesh>
FROM <http://togovar.biosciencedbc.jp/medgen>
WHERE {
 # Top レベル ('C') はノードじゃ無いため URI もラベルもないので objectList 出力の Attribute は取れるもので最上位を?mesh_childとする
{{#if top}}
  ?mesh_child meshv:treeNumber ?mesh_child_tree_number.
  ?mesh_child rdfs:label ?mesh_child_label.
  ?mesh_child_tree_number a meshv:TreeNumber .
  MINUS { 
    ?mesh_child_tree_number meshv:parentTreeNumber ?_parent . 
  }
  FILTER (CONTAINS(STR(?mesh_child_tree_number),"mesh/C"))
{{else}}
 # Top レベルでないノードは?mesh_inの直下?mesh_childを求める
  VALUES ?mesh_in {  <http://id.nlm.nih.gov/mesh/{{mesh_in}}> }

  # input mesh descriptor (D015470) to mesh children (D004915, D007946)
  GRAPH <http://togovar.biosciencedbc.jp/mesh>{
    ?mesh_in meshv:treeNumber ?mesh_in_tree_number .
    ?mesh_in rdfs:label ?mesh_in_label .
    ?mesh_child meshv:treeNumber/meshv:parentTreeNumber ?mesh_in_tree_number .
    ?mesh_child rdfs:label ?mesh_child_label.
    ?mesh_child meshv:treeNumber ?mesh_child_tree_number.
  }
{{/if}}

  # mesh childeren (D004915, D007946...) to medgen (C0023440, C0023461...)
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
    ?mesh_offsprings meshv:treeNumber/meshv:parentTreeNumber* ?mesh_child_tree_number .
  }
  
}
GROUP BY ?mesh_child ?mesh_child_label
ORDER BY ?mesh_child_tree
```

## `return`
- 整形
```javascript
({data})=>{
  const idVarName = "mesh";
  const idPrfix = "http://id.nlm.nih.gov/mesh/";
  const categoryPrefix = "http://id.nlm.nih.gov/mesh/";

  return data.results.bindings.map(d=>{
    let labelSets = new Set();
    return {
      categoryId: d.mesh_child.value.replace('http://id.nlm.nih.gov/mesh/',''),
//      catgoryTreeId: d.mesh_child_tree_number.value.replace('http://id.nlm.nih.gov/mesh/',''),
      label : labelSets.add(d.mesh_child_label.value).add(split('|',medgen_child_labels.value)),
      hasChild : d.mesh_offsprings_count.value
    }
  });
}
```
