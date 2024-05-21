# 2023_stiltgrass_migseq
Commands to process and analyze MigSeq data for stiltgrass greenhouse accessions


## 1. Rename files to remove extra useless characters

```bash
rename 's/^Cam_MigSeq_//' Cam_MigSeq_*
rename 's/_001.fastq.gz/.fastq.gz/' *.fastq.gz
rename 's/_L001//' *.fastq.gz
```

## 2. Replace first underscore with '-' which helps in downstream analysis (underscore is used as a delimiter in the strata file later on)

```bash
rename 's/_S/-S/' *.fastq.gz
```

## 3. move all reads to a new dir called 'reads' and set up new dirs for cleaned reads (with fastp) and fastqc/multiqc outputs

```bash
mkdir reads

mv *.fastq.gz reads

mkdir fastqc_raw
mkdir fastp
mkdir fastqc_clean
```


## 4. Run fastqc on everything, 30 threads (may take a while)

```bash
fastqc reads/*.fastq.gz -t 30 -o fastqc_raw
```

## 5. Run multiqc on fastqc raw output

```bash
multiqc fastqc_raw
```

## 6. Run fastp to trim adapters, poly-x tails, and low quality bases (sliding window = 16), keeping reads > 25 bp. 
##    Also trim off first 20 bp of all reads to remove the SSR region used for amplification

```bash
cd reads

for f1 in *_R1.fastq.gz
	do
        f2=${f1%%_R1.fastq.gz}"_R2.fastq.gz"
        fastp -i $f1 -I $f2 -w 16 -f 20 -F 20 --trim_poly_g --trim_poly_x -l 25 --cut_right -o "../fastp/fastp-$f1" -O "../fastp/fastp-$f2"
	done

-i, -I input fastq
-o, -O output fastq
--trim_poly_g --trim_poly_x -- trim poly g and poly (a/c/t) tails
-l minimum read L = 25
--cut_right = sliding window for trimming
```

## 7. Run fastqc on everything, 30 threads (may take a while)

```bash
fastqc fastp/*.fastq.gz -t 30 -o fastqc_clean
```

## 8. Run multiqc on fastqc raw output

```bash
multiqc fastqc_clean
```

## 9. Create a samples file

```bash
ls fastp/*_R1.fastq.gz > samples.txt

sed 's/_R1.fastq.gz//g' samples.txt > samples2.txt
```

## 10. Run the ISSRseq pipeline to index reference and further clean reads with bbduk

```bash

## First, unzip all of the read files

pigz -d fastp/*.fastq.gz &

## Then run
nohup ISSRseq_ReferenceBased.sh -O 2023_Oct_25_migseqtest -I fastp -S samples2.txt -R /data/cbarrett/angsd_analyses/js_reads_assoc_test/JS_allchr23.fasta -T 32 -M 25 -H 0 -P migseq_primers.fasta -X 35 &
```

## 11. Use the output dir to run ISSRseq_CreateBAMs pipeline, which maps the reads to reference with bbmap, sorts, and removes duplicates. Will need to change the timestamp part in -O to _2023_05_10_12_43 after the previous command is issued.

```bash
nohup ISSRseq_CreateBAMs.sh -O 2023_Oct_25_migseqtest_2023_10_25_13_53 -T 32 &
```

## 12. Use the output dir to run ISSRseq_CreateBAMs pipeline, which uses GATK to realign around indels, call, and filter SNPs

```bash
nohup ISSRseq_AnalyzeBAMs.sh -O 2023_Oct_25_migseqtest_2023_10_25_13_53 -T 32 -P 2 &
```
