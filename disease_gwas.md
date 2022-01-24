# Disease_GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `disease` Disease Name
  * default: Breast-ovarian cancer, familial 2

## Endpoint

{{ ep }}


## `result`
```sparql
DEFINE sql:select-option "order"

PREFIX tgv: <http://togovar.biosciencedbc.jp/variant/>
PREFIX oban: <http://purl.org/oban/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
PREFIX gwas_terms: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX oboInOwl: <http://www.geneontology.org/formats/oboInOwl#>

SELECT DISTINCT ?s ?p ?o2 #?p_value ?trait_name ?efo_uri ?rs ?dbsnp
#FROM <http://togovar.biosciencedbc.jp/variant>
#FROM <http://togovar.biosciencedbc.jp/ensembl37>
#FROM <http://togovar.biosciencedbc.jp/gwas-catalog>
#FROM <http://togovar.biosciencedbc.jp/efo>
FROM <http://togovar.biosciencedbc.jp/medgen>
WHERE {  
  
 ?s rdfs:label "{{disease}}";
    rdfs:seeAlso ?o;
    ?p ?o2.
#  FILTER ( strstarts(str(?o), "{{disease}}") ).
  
#  ?medgen_uri rdfs:label "{{disease}}" .
#    rdfs:seeAlso ?medgen_cid_uri.

#  ?medgen_cid_uri skos:definition ?definition ;
#    rdf:type medgen:ConceptID ;
#    dct:identifier ?concept_id.

#  ?medgen_cid_uri medgen:mgsat ?gene_location_node ;
#    medgen:mgsat ?gene_name_node ;
#    medgen:mgsat ?omim_node .
  
#  ?gene_location_node rdfs:label "NCBI_CYTOGEN_LOC" ;
#    rdf:value ?gene_location.

#  ?gene_name_node rdfs:label "GENESYMBOL" ;
#    rdf:value ?gene_name.

#  ?omim_node rdfs:label "NCBI_OMIM";
#    rdf:value ?omim.

  
  
  
  
#  VALUES ?variant { tgv:{{tgv_id}} }

#  GRAPH <http://togovar.biosciencedbc.jp/variant>{
#    ?variant rdfs:seeAlso ?dbsnp .
#    FILTER STRSTARTS(STR(?dbsnp), 'http://identifiers.org/dbsnp/')
#    BIND (REPLACE(STR(?dbsnp),'http://identifiers.org/dbsnp/','') as ?rs)
#    BIND (STRDT(REPLACE(STR(?dbsnp),'http://identifiers.org/dbsnp/',''), xsd:string) as ?rs_uri) 
#  }

#  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
#    ?gwas_snp rdfs:label ?rs_uri;
#      oban:is_subject_of ?trackable.

#    ?trackable oban:has_object ?efo_uri;
#      gwas_terms:has_p_value ?p_value;
#      gwas_terms:has_gwas_trait_name ?trait_name;
#      oban:has_object ?efo_uri.
#  }

#  GRAPH <http://togovar.biosciencedbc.jp/efo>{
#    ?efo_uri rdfs:label ?efo_label.
#    OPTIONAL { 
#       ?efo_uri oboInOwl:hasDbXref ?dbxref.
#    }
#    FILTER REGEX(?dbxref, "^MONDO:")
#  }
  
#  GRAPH <http://togovar.biosciencedbc.jp/mondo>{
#    OPTIONAL {
#      ?mondo oboInOwl:id ?dbxref;
#      rdfs:label ?mondo_label.
#    }
#  }

#  VALUES ?omim { {{omimid}} }

#?efo_uri ?p ?omim .
  
#?trackable oban:has_object ?efo_uri;
#   gwas_terms:has_p_value ?p_value;
#   gwas_terms:has_gwas_trait_name ?trait_name;
#   rdfs:label ?rs.

#?s1 ?p1 ?trackable.
#?s1 ?p2 ?mondo.
# FILTER (REGEX (?mondo , "http://purl.obolibrary.org/obo/MONDO" )).
#}
} limit 100
```


