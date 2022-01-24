# Gene_GWAS

## Parameters

* `ep` Endpoint
  * default: https://togovar-dev.biosciencedbc.jp/sparql
* `hgnc_id` HGNC ID
  * default: 404

## Endpoint

{{ ep }}


## `symbol2gwas`
```sparql
PREFIX faldo: <http://biohackathon.org/resource/faldo#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX gwas_terms: <http://rdf.ebi.ac.uk/terms/gwas/>
PREFIX oban: <http://purl.org/oban/>
PREFIX ro: <http://www.obofoundry.org/ro/ro.owl#>

SELECT DISTINCT ?ens_gene ?gwas_snp ?rs_uri ?ens_gene_start ?gwas_pos ?ens_gene_end ?trackable ?p_value ?trait_name ?efo_uri ?gwas_chr_num
WHERE {
  
  VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }
  ?hgnc_uri rdfs:label ?gene_symbol.
  

  GRAPH <http://togovar.biosciencedbc.jp/ensembl38>{
    ?ens_gene rdfs:label ?gene_symbol .
    ?ens_gene faldo:location / faldo:begin / faldo:position ?ens_gene_start.
    ?ens_gene faldo:location / faldo:end / faldo:position ?ens_gene_end.
    ?ens_gene faldo:location / faldo:end ?ens_gene_end_position.
   }

  ?gwas_snp gwas_terms:has_basepair_position ?gwas_pos;
    ro:located_in ?gwas_region.

  BIND( REPLACE( REPLACE( STR(?gwas_region), 'http://rdf.ebi.ac.uk/dataset/gwas/CytogeneticRegion/',''), '[q,p].*$', '') as ?gwas_chr_num).
  BIND( REPLACE( REPLACE( STR(?ens_gene_end_position), 'http:.*/GRCh38/',''), ':.*:.', '') as ?ens_chr_num).
   
  # The annotation available on our online search interface includes any Ensembl genes in which a SNP maps, or the closest upstream and downstream gene within 50kb .
  FILTER((?ens_gene_start - 50000) <= ?gwas_pos && (?ens_gene_end + 50000) >= ?gwas_pos).
  FILTER(?ens_chr_num = ?gwas_chr_num).

  GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog>{
    ?gwas_snp rdfs:label ?rs_uri;
      oban:is_subject_of ?trackable.

    ?trackable oban:has_object ?efo_uri;
      gwas_terms:has_p_value ?p_value;
      gwas_terms:has_gwas_trait_name ?trait_name.
  }
} ORDER BY ?p_value
```

## `rs_list`
```javascript

({symbol2gwas}) => {
  let prefix = "http://identifiers.org/dbsnp/";
  let literal = "^^<http://www.w3.org/2001/XMLSchema#string>";
  let result = "";
  let rs_array = {};

//  return symbol2gwas.results.bindings.map(x => x.rs_uri.value.replace(literal, "").replace(/^/, prefix)).join(" ");
 
  symbol2gwas.results.bindings.map(x =>{
    let rs = x.rs_uri.value;
    rs_array[rs] = rs.replace(literal, "").replace(/^/, prefix);
  })

  for (const [key, value] of Object.entries(rs_array)) {
    result += "<" + value + "> ";  
  }
  return result;
}
```


## `rs2tgv`
```sparql

PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX tgvo: <http://togovar.biosciencedbc.jp/ontology/>
PREFIX dct: <http://purl.org/dc/terms/>

SELECT DISTINCT ?variant ?rs
FROM <http://togovar.biosciencedbc.jp/variation>
WHERE {
  VALUES ?dbsnp { {{ rs_list }} }.
  
  ?s rdfs:seeAlso ?dbsnp;
    <http://purl.org/dc/terms/identifier> ?variant.

  
  BIND( REPLACE( STR(?dbsnp), 'http://identifiers.org/dbsnp/', '') as ?rs).
} 
```

## `result`

```javascript
({
  json({rs2tgv, symbol2gwas, rs_list}){
    let tgv_data = {};
    let gwas_data = {};

    rs2tgv.results.bindings.map(x => {
      let rs_id = x.rs.value;
      tgv_data[rs_id] = {
        variant: "<a href=\"/variant/" + x.variant.value + "\">" + x.variant.value + "</a>",
        rs: "<a href=\"https://www.ebi.ac.uk/gwas/variants/" + rs_id + "\">" + rs_id + "</a>"
      }
    });
  
     symbol2gwas.results.bindings.map(x => {
       let rs_id = x.rs_uri.value.replace("^^<http://www.w3.org/2001/XMLSchema#string>", "");
       let efo_link = x.efo_uri.value.replace("/efo/","/gwas/efotraits/")
       gwas_data[x.trackable.value]={   
         rs_id: rs_id,
         rs: "<a href=\"https://www.ebi.ac.uk/gwas/variants/" + rs_id + "\">" + rs_id + "</a>",
         position: x.gwas_chr_num.value + ":" + x.gwas_pos.value,
         p_value: x.p_value.value,
         trait_name: "<a href=\"" + efo_link + "\">" +  x.trait_name.value +  "</a>",
         trackable: x.trackable.value.replace("http://rdf.ebi.ac.uk/dataset/gwas/Trackable/","")
       }
     });
    
    return {
      columns: [["tgvid"], ["rs"], ["variant_position"], ["p-value"], ["traitname"],["trackable"]],
      data: Object.keys(gwas_data).map(function(key){
        let gwas = gwas_data[key];
        let tgv = tgv_data[gwas.rs_id] || {};
        return [tgv.variant, gwas.rs, gwas.position, gwas.p_value, gwas.trait_name, gwas.trackable]
    })
      
      
    };
  }
})
```
