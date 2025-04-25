
# lncRNA-seq Analysis Pipeline (Diploid vs Tetraploid Brassica oleracea)

This repository documents a full pipeline for lncRNA-seq analysis comparing diploid and tetraploid **Brassica oleracea**. The pipeline includes quality control, trimming, alignment, transcript assembly, and quantification.

---

## 📁 Directory Structure

```bash
.
├── raw_data/
├── qc_reports/
├── trimm_output/
├── postfastqc_output/
├── align_out/
├── bam_files/
├── transcript_assembly/
├── featurecounts_output/
└── ref_genome/
```

---

## 🧬 Step 1: Quality Control (FastQC)

Run FastQC for all paired-end reads and save reports separately.

```bash
for file in *_1.fq.gz; do
    pair="${file/_1.fq.gz/_2.fq.gz}"
    prefix=$(basename "$file" | sed 's/_1\.fq\.gz//')
    mkdir -p qc_reports/"$prefix"
    fastqc "$file" "$pair" -o qc_reports/"$prefix"/
done
```

---

## ✂️ Step 2: Trimming Reads (Trimmomatic)

Remove adapters and low-quality bases.

```bash
mkdir -p trimm_output

for file in *_1.fq.gz; do
    pair="${file/_1.fq.gz/_2.fq.gz}"
    prefix=$(basename "$file" | sed 's/_1\.fq\.gz//')
    trimmomatic PE -phred33 \
        "$file" "$pair" \
        trimm_output/"${prefix}_trimmed_1.fq.gz" trimm_output/"${prefix}_unpaired_1.fq.gz" \
        trimm_output/"${prefix}_trimmed_2.fq.gz" trimm_output/"${prefix}_unpaired_2.fq.gz" \
        ILLUMINACLIP:TruSeq3-PE.fa:2:30:10 HEADCROP:10 SLIDINGWINDOW:4:20 MINLEN:50
done
```

---

## ✅ Step 3: Post-trimming QC (FastQC)

Check the quality of trimmed reads.

```bash
mkdir -p postfastqc_output

for file1 in trimm_output/*_trimmed_1.fq.gz; do
    file2="${file1/_trimmed_1.fq.gz/_trimmed_2.fq.gz}"
    fastqc "$file1" "$file2" -o postfastqc_output/
done
```

---

## 🧬 Step 4: Read Alignment (HISAT2)

Build HISAT2 index and align trimmed reads.

```bash
# Build index (only once)
hisat2-build Brassica_oleracea_JZS_v2.fasta Brassica_oleracea_index

# Align
mkdir -p align_out

for file1 in trimm_output/*_trimmed_1.fq.gz; do
    file2="${file1/_trimmed_1.fq.gz/_trimmed_2.fq.gz}"
    prefix=$(basename "$file1" | sed 's/_trimmed_1\.fq\.gz//')
    hisat2 -p 20 -x /path/to/Brassica_oleracea_index \
           -1 "$file1" -2 "$file2" \
           -S align_out/"${prefix}.sam"
done
```

---

## 🧪 Step 5: Convert & Sort BAM Files (SAMtools)

```bash
mkdir -p bam_files

for sam_file in align_out/*.sam; do
    prefix=$(basename "$sam_file" .sam)
    samtools view -bS "$sam_file" | samtools sort -@ 20 -o bam_files/"${prefix}_sorted.bam"
    samtools index -@ 20 bam_files/"${prefix}_sorted.bam"
done
```

---

## 🧾 Step 6: Convert GFF3 to GTF

```bash
gffread Brassica_oleracea.var_capitata.JZS.v2.gene.gff3 -T -o Brassica_oleracea.var_capitata.JZS.v2.gene.gtf
```

---

## 🧬 Step 7: Transcript Assembly (StringTie)

```bash
mkdir -p transcript_assembly

for bam_file in bam_files/*.bam; do
    prefix=$(basename "$bam_file" .bam)
    stringtie "$bam_file" -o transcript_assembly/"${prefix}_transcripts.gtf" \
              -p 8 -G /path/to/Brassica_oleracea.var_capitata.JZS.v2.gene.gtf
done
```

---

## 🧮 Step 8: Gene Expression Quantification (featureCounts)

### (a) Separate count files for each BAM

```bash
mkdir -p featurecounts_output

for BAM in bam_files/*.bam; do
    BASENAME=$(basename "$BAM" .bam)
    featureCounts -T 8 -a ref_genome/Brassica_oleracea.var_capitata.JZS.v2.gene.gtf \
                  -o featurecounts_output/"${BASENAME}_counts.txt" \
                  -g gene_id -t transcript -p -s 0 "$BAM"
done
```

### (b) Combined count file for all BAMs

```bash
featureCounts -T 8 \
    -a ref_genome/Brassica_oleracea.var_capitata.JZS.v2.gene.gtf \
    -o featurecounts_output/combined_counts.txt \
    -g gene_id -t transcript -p -s 0 bam_files/*.bam
```

---

## 📌 Requirements

- FastQC
- Trimmomatic
- HISAT2
- SAMtools
- GFFread
- StringTie
- Subread (featureCounts)

---

## 🔖 Notes

- Replace `/path/to/` with your actual directories.
- Check if strand-specific flag `-s 0` in `featureCounts` is correct for your library type.

---

## 🧑‍🔬 Author

Shima Mahmoudi  
lncRNA expression pipeline – Diploid vs Tetraploid Brassica oleracea

---
