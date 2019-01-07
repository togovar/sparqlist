# ClinVar Conditions

## Parameters

* `accession` RefSeq sequence accession
  * default: NC_000012.11
* `position` Sequence position
  * default: 112241766
* `ref` Reference sequence (use - for insertion varition)
  * default: G
* `alt` Alternative sequence (use - for deletion varition)
  * default: A

### accession

| Chromosome | Accession |
|----|--------------|
| 1  | NC_000001.10 |
| 2  | NC_000002.11 |
| 3  | NC_000003.11 |
| 4  | NC_000004.11 |
| 5  | NC_000005.9  |
| 6  | NC_000006.11 |
| 7  | NC_000007.13 |
| 8  | NC_000008.10 |
| 9  | NC_000009.11 |
| 10 | NC_000010.10 |
| 11 | NC_000011.9  |
| 12 | NC_000012.11 |
| 13 | NC_000013.10 |
| 14 | NC_000014.8  |
| 15 | NC_000015.9  |
| 16 | NC_000016.9  |
| 17 | NC_000017.10 |
| 18 | NC_000018.9  |
| 19 | NC_000019.9  |
| 20 | NC_000020.10 |
| 21 | NC_000021.8  |
| 22 | NC_000022.10 |
| X  | NC_000023.10 |
| Y  | NC_000024.9  |

## Endpoint

https://togovar.org/sparql

## `result` fetch transcripts

```sparql
DEFINE sql:select-option "order"

PREFIX cvo: <http://purl.jp/bio/10/clinvar/>
PREFIX vcv: <http://identifiers.org/clinvar:>
PREFIX faldo: <http://biohackathon.org/resource/faldo#>

SELECT DISTINCT ?vcv ?rcv ?condition ?interpretation ?submission_count ?review_status ?last_evaluated
FROM <http://togovar.org/graph/clinvar>
WHERE {
  ?vcv a cvo:VariationArchiveType ;
    cvo:interpreted_record/cvo:haplotype?/cvo:simple_allele/cvo:location/faldo:location ?_location .
  ?_location cvo:assembly "GRCh37" ;
    cvo:accession "{{accession}}" ;
    cvo:start {{position}} ;
    cvo:reference_allele "{{ref}}" ;
    cvo:alternate_allele "{{alt}}" .

  ?vcv cvo:interpreted_record/cvo:rcv_list/cvo:rcv_accession ?_rcv .
  ?_rcv cvo:interpretation ?interpretation ;
    cvo:accession ?rcv ;
    cvo:review_status ?review_status ;
    cvo:submission_count ?submission_count ;
    cvo:date_last_evaluated ?last_evaluated ;
    cvo:interpreted_condition/cvo:type_rcv_interpreted_condition ?condition .
}
```

