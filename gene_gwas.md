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
#SELECT DISTINCT ?rs_uri
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
#} ORDER BY ?rs_uri
} ORDER BY ?p_value
```

## `rs_list`
```javascript

({symbol2gwas}) => {
  let prefix = "http://identifiers.org/dbsnp/";
  let literal = "^^<http://www.w3.org/2001/XMLSchema#string>";
  let result = "";
  let rs_array = {};

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


## `rs2associations_api`
```javascript
async ({symbol2gwas}) => {
  const literal = "^^<http://www.w3.org/2001/XMLSchema#string>";
  const options = { method: 'GET'};
  var rs_array = {};
  var response = [];
  var result = {};
  
  symbol2gwas.results.bindings.forEach((x) => {
    let rs = x.rs_uri.value.replace(literal, "");
    rs_array[rs] = rs;
  });
  
  for (const [key, value] of Object.entries(rs_array)) {
    let api_url ="https://www.ebi.ac.uk/gwas/rest/api/singleNucleotidePolymorphisms/"+ value +"/associations";
    response.push( fetch(api_url, options).then(res=>res.json()));
  }
  var api_res = await Promise.all(response);

  for (var i=0; i < api_res.length; i++){
    var associations = api_res[i]["_embedded"]["associations"];

    for(var j=0; j < associations.length; j++){
        // todo stanzaで表示する項目をresultに設定すること
      let trackable = associations[j]._links.self.href.replace("https://www.ebi.ac.uk/gwas/rest/api/associations/","");

      for (var k=0; k < associations[j].loci.length; k++)[
        
        }
      
      if(!result[trackable]){
        result[trackable] = associations[j];
      } else {
        result["associations_error"] = "trackable is not primary" ;
      }
    }
  }
  return result;
}
```

## `rs2study_api`
```javascript
async ({symbol2gwas}) => {
  const literal = "^^<http://www.w3.org/2001/XMLSchema#string>";
  const options = { method: 'GET'};
  var rs_array = {};
  var response = [];
  var result = {};
  
  symbol2gwas.results.bindings.forEach((x) => {
    let rs = x.rs_uri.value;
    rs_array[rs] = rs.replace(literal, "");
  })
  
  for (const [key, value] of Object.entries(rs_array)) {
    let api_url ="https://www.ebi.ac.uk/gwas/rest/api/singleNucleotidePolymorphisms/"+ value +"/studies";
    response.push( fetch(api_url, options).then(res=>res.json()));
  }
  var api_res = await Promise.all(response);

  for (var i=0; i < api_res.length; i++){
    var studies = api_res[i]["_embedded"]["studies"];

    for(var j=0; j < studies.length; j++){
      var pubmedId = studies[j].publicationInfo.pubmedId;
      var accession = studies[j].accessionId;
      var ancestries = studies[j].ancestries.type;

      var study_data = {};
      study_data["pubmedId"] = studies[j].publicationInfo.pubmedId;
      study_data["accessionId"] = studies[j].accessionId;
      study_data["initialSampleSize"] = studies[j].initialSampleSize;
      study_data["ancestries"] = studies[j].ancestries.type;
      study_data["ReplicationSampleDescription"] = studies[j].replicationSampleSize;
      
      for (var k=0; k < studies[j].ancestries.length; k++){
        var ancestry = studies[j].ancestries[k].numberOfIndividuals+ " "+ studies[j].ancestries[k].ancestralGroups[0]["ancestralGroup"];  
        if (studies[j].ancestries[k].type == "initial"){
          study_data["DiscoverySampleAncestry"] = ancestry;
        } else if (studies[j].ancestries[k].type == "replication"){
          // type=replicationは複数個あるので末尾に追加する方式でいいか要確認
          if (!study_data["ReplicationSampleAncestry"]){
            study_data["ReplicationSampleAncestry"] = ancestry;
          }else{
            study_data["ReplicationSampleAncestry"] += "," + ancestry; 
          }
        }
      }
      
/*
      if(!result[pubmedId]){
        // todo stanzaで表示する項目をresultに設定すること
        result[pubmedId] = studies[j];
      }
*/
      if(!result[accession]){
        // todo stanzaで表示する項目をresultに設定すること
        result[accession] = study_data;
        
      } else {
//        result["studies_error:" + study_accession] = "study accession is not primary" ;
      }
    }
  }
  return result;
}
```

## `studies_from_rdf`
```sparqlist
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX oa: <http://www.w3.org/ns/oa#>
PREFIX bibo: <http://purl.org/ontology/bibo/>
PREFIX dcterms: <http://purl.org/dc/terms/>

SELECT DISTINCT ?study ?accessionId ?pmid
WHERE {
    VALUES ?hgnc_uri { <http://identifiers.org/hgnc/{{ hgnc_id }}> }
 
    GRAPH <http://togovar.biosciencedbc.jp/hgnc>{ 
      ?hgnc_uri rdfs:seeAlso ?ncbi_gene.
      ?ncbi_gene rdf:type  <http://identifiers.org/ncbigene>.
    }
    GRAPH <http://togovar.biosciencedbc.jp/pubtator>{
      ?pubtator_node rdf:type oa:Annotation ;
       oa:hasBody ?ncbi_gene;
       oa:hasTarget ?pmid_uri.
    } 
    GRAPH <http://togovar.biosciencedbc.jp/pubmed>{
      ?pmid_uri dcterms:source ?journal ;
      dcterms:title ?title ;
      dcterms:issued ?year ;
      bibo:pmid ?pmid .
   }
    GRAPH <http://togovar.biosciencedbc.jp/gwas-catalog-studies>{
      ?study gwas:has_pubmed_id ?pmid;
        dct:identifier ?accessionId.
    }
}

```
## `rs2tgv`
```sparql

PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>

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
        variant_disp:  x.variant.value,
        variant_link:  "/variant/" + x.variant.value,
        rs_disp: rs_id,
        rs_link: "https://www.ebi.ac.uk/gwas/variants/" + rs_id
      }
    });
  
    symbol2gwas.results.bindings.map(x => {
      let rs_id = x.rs_uri.value.replace("^^<http://www.w3.org/2001/XMLSchema#string>", "");
      let efo_link = x.efo_uri.value.replace("/efo/","/gwas/efotraits/")
      gwas_data[x.trackable.value]={   
        rs_disp: rs_id,
        rs_link: "https://www.ebi.ac.uk/gwas/variants/" + rs_id,
        position: x.gwas_chr_num.value + ":" + x.gwas_pos.value,
        p_value: x.p_value.value,
        trait_name: x.trait_name.value,
        efo_link:  efo_link,
        trackable: x.trackable.value.replace("http://rdf.ebi.ac.uk/dataset/gwas/Trackable/","")
      }
    });
    
    let data = [];
    Object.keys(gwas_data).map(function(key){
      let gwas = gwas_data[key];
      let tgv = tgv_data[gwas.rs_disp] || {};
      let obj = {
        variant_disp: tgv.variant_disp,
        variant_link: tgv.variant_link,
        rs_disp: tgv.rs_disp,
        rs_link: tgv.rs_link,
        position: gwas.position,
        p_value: gwas.p_value,
        trait_name:gwas.trait_name,
        efo_link:  gwas.efo_link,
        trackable: gwas.trackable
      };
      data.push(obj);
    });
    return data;
  }
})
```
