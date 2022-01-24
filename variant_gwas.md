# Variant_GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `tgv_id` TogoVar ID
  * default: tgv47264307

## Endpoint

{{ ep }}

## `tgv2gwas`
```sparql
#DEFINE sql:select-option "order"

#PREFIX tgv: <http://togovar.biosciencedbc.jp/variation/>
PREFIX oban: <http://purl.org/oban/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
PREFIX gwas_terms: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX oboInOwl: <http://www.geneontology.org/formats/oboInOwl#>

SELECT DISTINCT ?variant ?rs ?p_value ?trait_name ?rs_link ?efo_link
#FROM <http://togovar.biosciencedbc.jp/variation>
#FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
#FROM <http://togovar.biosciencedbc.jp/efo>
WHERE {  
  
  GRAPH <http://togovar.biosciencedbc.jp/variation>{
#  VALUES ?variant { tgv:{{tgv_id}} }
#    ?variant rdfs:seeAlso ?dbsnp .
    
    VALUES ?variant { "{{tgv_id}}" }.
    ?s rdfs:seeAlso ?dbsnp;
      <http://purl.org/dc/terms/identifier> ?variant.
    
    FILTER STRSTARTS(STR(?dbsnp), 'http://identifiers.org/dbsnp/')
    BIND (REPLACE(STR(?dbsnp),'http://identifiers.org/dbsnp/','') as ?rs)
    BIND (STRDT(REPLACE(STR(?dbsnp),'http://identifiers.org/dbsnp/',''), xsd:string) as ?rs_uri) 
    BIND (REPLACE(STR(?dbsnp),'http://identifiers.org/dbsnp/','https://www.ebi.ac.uk/gwas/variants/') as ?rs_link)
  }

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
    ?gwas_snp rdfs:label ?rs_uri;
      oban:is_subject_of ?trackable.
  }
    
    ?trackable oban:has_object ?efo_uri;
      gwas_terms:has_p_value ?p_value;
      gwas_terms:has_gwas_trait_name ?trait_name;
      oban:has_object ?efo_uri.
    ?efo_uri rdfs:label ?efo_label.
    BIND (REPLACE(STR(?efo_uri),'http://www.ebi.ac.uk/efo/','https://www.ebi.ac.uk/gwas/efotraits/') as ?efo_link)

#    OPTIONAL { 
#      ?efo_uri oboInOwl:hasDbXref ?dbxref.
#      FILTER REGEX(?dbxref, "^MONDO:")
#    }
#  
#    GRAPH <http://togovar.biosciencedbc.jp/mondo>{
#      OPTIONAL {
#        ?mondo oboInOwl:id ?dbxref;
#          rdfs:label ?mondo_label.
#      }
#    }
#    VALUES ?omim { {{omimid}} }
#
#    ?efo_uri ?p ?omim .
#    ?trackable oban:has_object ?efo_uri;
#       gwas_terms:has_p_value ?p_value;
#       gwas_terms:has_gwas_trait_name ?trait_name;
#       rdfs:label ?rs.
#
#    ?s1 ?p1 ?trackable.
#    ?s1 ?p2 ?mondo.
#    FILTER (REGEX (?mondo , "http://purl.obolibrary.org/obo/MONDO" )).
  
  BIND (STRDT(REPLACE(STR(?rs),'rs',''), xsd:integer) as ?sort_number).

} ORDER BY ?sort_number
```

## `result`
```javascript

({
  json({tgv2gwas}){
    let gwas_data = {};
    let id = 0;
    tgv2gwas.results.bindings.map(x => {
      id++;
      gwas_data[id]={
//        variant: x.variant.value,
        disp_rs: "<a href=\"" + x.rs_link.value + "\">" + x.rs.value + "</a>",
//        position: "",
        p_value: x.p_value.value,
        disp_trait_name: "<a href=\"" + x.efo_link.value + "\">" +  x.trait_name.value +  "</a>"
      }
    });

    return {
//      columns: [["tgvid"], ["rs"], ["variant_position"], ["p-value"], ["traitname"]],
      columns: [["rs"], ["p-value"], ["traitname"]],
      data: Object.keys(gwas_data).map(function(key){
        let gwas = gwas_data[key];
        return [gwas.disp_rs, gwas.p_value, gwas.disp_trait_name]
      })
    }
  }
})
```



