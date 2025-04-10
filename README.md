# PathoSV
PathoSV: A transcriptome-aware tool for assessing Structural Variant pathogenicity.


PathoSV estimates Structural Variant (SV) pathogenicity using a 'truncated ratio' that quantifies predicted gene transcription disruption in a specific tissue relative to its TPM. Input SV coordinates and tissue; PathoSV outputs the ratio (flagging > 0.25 as potentially pathogenic), SV population frequency, affected gene annotations (OMIM, GO), and offers deepseek interpretation for relating SV impact to a specified disease.

PathoSV consists of two components: SV frequency annotation and SV pathogenicity annotation. The frequency annotation script provides the frequency of target SVs in natural populations, helping to filter rare variants. The pathogenicity annotation script evaluates the pathogenicity of variants and genes from four dimensions:1. Genomic annotation;2. Genomic variant constraint scores;3. Transcriptome annotation;4. Pathogenic gene and variant information. These are two independent scripts, and you can use different script combinations as needed.

## Reference files introduction
SV frequency annotation script use the SVs identified in the natural population cohorts as the background SV set to perform frequency annotation on the submitted SVs. We recommend using a natural population cohort of the same ethnicity as the study cohort as the background SV set, to avoid misclassifying ethnicity-specific common SVs as rare variants. 

The pathogenicity annotation script use 5 reference files：1. Genomic Annotation: The annotation file contains gene positional information, disease associations, and biochemical function data, sourced from the GENCODE GTF, OMIM database, and GO database respectively；2. Exon Annotation: The annotation file contains exon position and corresponding transcript information, sourced from the GENCODE GTF; 3. Genomic Variant Constraint Score:
A Gnocchi score >4 indicates variant constraint in the region, suggesting that variants within this interval are more likely to be deleterious; 4. Transcriptome Annotation: The transcript TPM data is used to calculating its transcript disruption ratio in relevant tissues, with a ratio >0.25 considered pathogenic. (Truncated_ratio = ∑(TPM of SV-overlapped transcripts) / ∑(Gene TPM)); 5. Known Pathogenic Variant Annotation: Known pathogenic SVs are annotated based on ClinVar records.

We provide 1 background SV allele frequency annotation file and 5 reference files for SV pathogenicity annotation. You may also use your own reference files in the same format.
1. 1KG_china_gtex_manta_qc_jasmine_merge0.8_max_af.txt：SV AF set of 196 1KGP CHB+CHS cohort and 838 GTEx cohort, when a SV appears in both cohorts, retain the one with the highest AF value.
2. gene_gencodev26_OMIM_GO_info.txt：This annotation file contains gene positional information, disease associations, and biochemical function data, sourced from the GENCODE V26 hg38 GTF, OMIM database, and GO database respectively.
3. gencode.v26.annotation_exon_info.txt：This annotation file contains exon position and corresponding transcript information, sourced from the GENCODE  V26 hg38 GTF.
4. constraint_z_genome_1kb.qc.download.txt.gz：Gnocchi is genomic non-coding constraint of haploinsufcient variation.You can download this file from https://gnomad.broadinstitute.org/downloads#v4-constraint.
5. 55tissues_p10_v26_transcript_tpm_mean.txt：Transcripts mean TPM in 55 tissues, which was calculated from GTEx dataset of 54 tissues and one retina dataset.
6. clinvar_20241027_sv_info.txt：Known pathogenic infomation from Clinvar database.

## Run PathoSV
### 1. SV frequency annotation
```
python /bin/sv_background_af_annotation.py [-h] [--spid] [--threshold] [--rare] --backgound  --sv  -o
```

#### Input file:
- backgound: The background SV set to perform frequency annotation on the submitted SVs.
- sv: The submitted SV file must contain at least 4 columns (CHROM, START, END, SVTYPE) without column headers and may include sample genotype information, with sample IDs to be provided via the "--spid" parameter.
- o: Output file prefix, including the absolute path and filename prefix.
  
#### Parameters:
- spid: If the input SV file contains genotype information, sample IDs can be provided through this parameter, with one sample ID per line.
- threshold: The cutoff value for filtering rare variants (default: 0.01).
- rare: Whether to output results containing only rare SVs (default: "True").

#### Output files:
1. [output]_background_af_annotation.txt：All SV frequency annotation result, added the "Background_AF" column behind "SVTYPE" column.
2. [output]_background_af_annotation_rare.txt：Rare SVs set.

### 2. SV pathogenicity annotation
```
python /bin/sv_pathogenic_annotaion.py [-h] [--gene] [--exon] [--gnocchi] [--clinvar] [--tissue] [--tpm_trans] --sv  -o
```

#### Input file:
- sv: The submitted SV file must contain at least 4 columns (CHROM, START, END, SVTYPE) with column headers.
- o: Output file prefix, including the absolute path and filename prefix.

#### Parameters:
- gene：Genomic Annotation file, with CHROM, START, END, ENSG, GENE_TYPE, SYMBOL, MIM, Phenotypes, GO_MF, GO_BP, GO_CC 11 columns.
- exon：Exon Annotationfile, with CHROM, START, END, CLASS, ENSG, ENST, GENE_TYPE, SYMBOL, TRANS_TYPE 9 columns.
- gnocchi：Gnocchi score file；
- clinvar：Known pathogenic infomation from Clinvar database, with CHROM, START, END, SVTYPE, SVLEN, CLNSIG, CLNDN, ALLELEID 8 columns.
- tissue：Disease associated tissue, default is "None". If the value is "None", SV transcriptome annotation will not be performed.
- tpm_trans：Transcripts mean TPM matrix which contain tissue in parameter "--tissue" at least.

*If you want to use the default annotation files, make sure that "ref_dir" is in your work folder.*

#### Output columns:
- SYMBOL: Gene symbol.
- ENSG: Gene ENSG ID.
- GENE_TYPE: Gene type in GENCODE.
- ENST: Transcrpt ENST ID.
- TRANS_TYPE: Transcrpt type in GENCODE.
- Consequence: Position of SV in gene, which contains "Overlap all gene"，"Overlap exon"，"Overlap intron"，"Overlap intergenic" 4 types.
- Gnocchi: Max gnocchi score in SV region. Gnocchi > 4 means SV covers conservative regions.
- MIM: OMIM ID.
- Phenotypes: Gene corralation disease and Inheritance of gene in OMIM database.
- GO_MF: Gene molecular function in GO database.
- GO_BP: Gene biological process in GO database.
- GO_CC: Gene cellular component in GO database.
- ClinVar_CLNSIG:Pathogenic SV lable in ClinVar database.
- ClinVar_CLNDN: Pathogenic SV corralation disease in ClinVar database.
- Sum_truncated_trascript_tpm: Total transcripts TPM overlaped by SV.	
- Gene_tpm: Gene TPM；
- Truncated_ratio: ∑(TPM of SV-overlapped transcripts) / ∑(Gene TPM). Truncated_ratio >0.25 means pathogenic.
- Top_truncated_trascript_TPM_rank: The top TPM of SV-overlapped transcripts in all transcripts of gene.

#### Output files:
- If transcriptom annotation is used, the output file will be [output]\_annotation\_[tissue]_transcript_tpm_truncated_percent.txt. Each row records the information of SV-GENE, without "ENST" and "TRANS_TYPE" columns.
- If transcriptom annotation isn't used, the output file will be [output]_annotation.txt，Each row records the information of SV-TRANS，without "Sum_truncated_trascript_tpm", "Gene_tpm" and "Truncated_ratio" columns.
