# Exonerate Gene Extraction Pipeline (EXONEXT.py)

## Overview

This repository contains a Python pipeline for processing [Exonerate](https://www.ebi.ac.uk/about/vertebrate-genomics/software/exonerate) alignment results to identify, extract and characterise candidate gene sequences.

The pipeline was developed for comparative analysis of autosomal (ACHR) and germline-restricted chromosome (GRC) paralogs in songbirds.

It processes Exonerate output files, selects relevant alignment hits, extracts coding sequences, evaluates sequence completeness and identifies potential pseudogenisation events. It can also retrieve intron positions and sizes from the original Exonerate alignments.

## Main functionalities

The pipeline consists of the following steps:

1. **Alignment filtering**
   - Reads Exonerate output (`.out`) files, resulting from an exonerate run (exonerate --model protein2genome --percent 50 -q "$QUERY_FILE" -t "$GENOME_FILE" --showcigar yes --showquerygff yes --minintron 10 --ryo "Target %ti (%tab-%tae)\n%tas\n" > "$OUTPUT_FILE")
   - Optionally filters alignment hits according to a user-defined raw score threshold.

2. **CIGAR extraction**
   - Extracts CIGAR alignment information.
   - Records the species, gene, paralog, query coordinates, scaffold, genomic coordinates, reading frame and alignment score.
   - Generates a comprehensive table of alignment results.

3. **Selection of candidate sequences**
   - **ACHR:** Selects the highest-scoring alignment for each gene and species combination.
   - **GRC:** Selects alignments matching annotated GRC transcripts using transcript identifiers, scaffold, reading frame, alignment score and genomic start position.
   - Extracts the corresponding alignment regions for downstream sequence reconstruction.

4. **Coding sequence extraction**
   - Reconstructs nucleotide sequences from the selected Exonerate alignments.
   - Generates individual FASTA files for each selected sequence.

5. **Sequence completeness assessment**
   - Compares the extracted sequence length against the corresponding query sequence.
   - Calculates sequence completeness as a percentage of the query length.

6. **Detection of potential pseudogenisation**
   - Identifies premature stop codons.
   - Registers frameshift mutations, insertions and deletions.
   - Evaluates potential pseudogenisation based on the presence of stop codons in the reconstructed sequences.
   - Records the positions of detected mutations relative to the query sequence.

7. **Intron characterisation (optional)**
   - Retrieves intron positions and sizes from Exonerate alignment fragments.
   - Records intron information for individual alignments.
   - Integrates intron information with the selected sequence and pseudogene results.

## Requirements

The pipeline requires Python and the following packages:

- Python 3
- [Biopython](https://biopython.org/)
- [pandas](https://pandas.pydata.org/)
- [openpyxl](https://openpyxl.readthedocs.io/)
- [XlsxWriter](https://xlsxwriter.readthedocs.io/)

## Input files and directory structure

The pipeline expects the following directory structure:

```text
working_directory/
│
├── EXONEXT.py
│
├── Exonerate_output_all/
│   ├── *.out
│
├── Query_all/
│   ├── gene1.fas
│   ├── gene2.fas
│   └── ...
│
└── 0-assemblies_annotations/
    ├── *.gtf
    └── ...
```

### Input descriptions

| Input | Description |
|---|---|
| Exonerate `.out` files | Raw Exonerate alignment output containing alignment scores, CIGAR strings and alignment information. |
| `.hit` files | Filtered Exonerate output containing alignments above the selected raw score threshold. |
| `Query_all/` | FASTA files containing the reference query sequences used in the Exonerate searches. Files are named according to the gene. |
| `0-assemblies_annotations/` | GTF annotation files containing transcript coordinates and identifiers used to select annotated GRC alignments. |

GTF files must contain transcript features with `gene_id` and `gene_name` attributes.

## Configuration

The main configuration parameters are defined near the beginning of the Python script.

| Parameter | Description | Default |
|---|---|---|
| `direct` | Directory containing Exonerate output files. | `Exonerate_output_all` |
| `method` | Method used to process alignment output files. | `2` |
| `threshold` | Minimum raw alignment score retained when filtering hits. | `200` |
| `intron_input` | Whether intron positions and sizes are retrieved. | `Y` |

### Alignment processing methods

Three processing methods are available:

- **Method 1 – Raw output files:** Processes all original Exonerate `.out` files.
- **Method 2 – Filtered hit files:** Extracts alignments with raw scores equal to or above the specified threshold and processes the resulting `.hit` files.
- **Method 3 – Combined processing:** Processes both `.out` and `.hit` files. This method is intended for combining raw output from selected files with filtered results from other files. Existing `.hit` files must be available.

Method 2 is generally faster and reduces processing time, but some CIGAR entries may not be registered in the resulting output.

## Output files

The main outputs include:

- `fasta_sequences/` – retrieved FASTA sequences.
- `Exonerate_output_all_final_table.xlsx` – Excel workbook containing three worksheets:

### 1. `final_table`

Contains the selected ACHR and GRC sequences and their associated characterisation results, including:

- Species and gene identifiers.
- Paralog classification (ACHR or GRC).
- Selected transcript identifiers.
- Alignment coordinates and scores.
- Sequence completeness.
- Stop codon positions and counts.
- Frameshift positions and counts.
- Insertion and deletion positions.
- Functional classification (`functional` or `pseudogene`).
- Potential pseudogene classification.

### 2. `introns`

Contains intron information in a long-format table, with one row per intron. Includes:

- Species and gene.
- Exonerate output file and query identifier.
- Scaffold and genomic alignment coordinates.
- Alignment score.
- Intron number.
- Intron position relative to the query sequence.
- Intron size in base pairs.
- Total number of introns in the alignment.

### 3. `pseudogenes`

Contains a long-format table of detected stop codon, frameshift and deletion positions, with one row per recorded mutation site.

### Additional intermediate outputs

During execution, the pipeline also generates intermediate Excel files and a directory:

- `cuts/` – extracted Exonerate alignment regions.
- `*_cigar_lines.xlsx` – extracted CIGAR information.
- `*_selected_sequences.xlsx` – selected alignment records.
- `*_selected_sequences_completeness.xlsx` – sequence completeness information.
- `*_selected_sequences_final.xlsx` – selected sequences with pseudogene characterisation.
- `*_intron_information.xlsx` – intron information in list format.
- `*_intron_information1.xlsx` – intron information in long format.

Most intermediate Excel files are removed automatically after successful generation of the final workbook.

## Notes and limitations

- The selection of ACHR sequences is based on the highest alignment score for each gene and species combination.
- GRC sequence selection depends on the availability and consistency of transcript annotations in the supplied GTF files.
- GRC alignment selection uses matching transcript identifiers, scaffold, reading frame, alignment score and genomic start position.
- Sequence completeness is calculated relative to the corresponding query sequence.
- Potential pseudogenisation is assessed through the detection of stop codons and frameshift-associated features in the reconstructed alignments.
- The intron extraction procedure uses the original Exonerate output files.
