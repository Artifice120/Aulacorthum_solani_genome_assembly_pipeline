# Aulacorthum_solani_genome_assembly_pipeline
Commands used for assembly and annotation of A. solani reads


# Canu: assemble reads to contigs
```
canu -p foxglove -d foxgloves genomeSize=300m /lustre/isaac/scratch/jtorre28/basecalling/A_solani/aulacorthum_nanopore.fastq utgReAlign=true overlapper=mhap usegrid=false
```
# Purge haplotigs

## Pre-Prep: sanatize/format raw nanopore reads for purge haplotigs program
```
seqkit sana  /lustre/isaac/scratch/jtorre28/basecalling/A_solani/aulacorthum_nanopore.fastq > /lustre/isaac/scratch/jtorre28/basecalling/A_solani/aulacorthum_sanatized.fasta
```
## Prep: align reads against contigs
```
minimap2 -t 48 -ax map-ont /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/polished2.fasta /lustre/isaac/scratch/jtorre28/basecalling/A_solani/aulacorthum_sanatized.fasta --secondary=no | samtools sort -m 1G -o /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/aligned2.bam -T /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/tmp2.ali
```
## Plot: build coverage histogram
```
purge_haplotigs  hist  -b  /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/aligned2.bam -g /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/polished2.fasta -t 48
```
## Set: Use histogram to select low (-l) mid (-m) and high (-h) "valley" points
```
purge_haplotigs  cov  -i /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/aligned2.bam.200.gencov -l 5  -m 15  -h 60  -o /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/coverage_stats_pilon2_foxglove.csv -j 60  -s 60
```
## Purge haplotigs
```
purge_haplotigs  purge  -g  /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/polished2.fasta -c /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/coverage_stats_pilon2_foxglove.csv -t 45
```
# Pilon Polishing with read coverage from Short Illumina reads as well as the original coverage from nanopore reads.
```
java -Xmx90G -jar /lustre/isaac/scratch/jtorre28/enviro/pilonn/share/pilon-1.24-0/pilon.jar --genome /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/polished.fasta --frags /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/illumina-polished1.bam --frags /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/illumina-polished2.bam --output polished2 --changes
```
## Visualize coverage of contigs by illumina short reads ( Used to collapse contigs labeled as bubbles by canu ).
```
samtools coverage /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/illumina-polished1.bam > illumina-coverage-foxglove
```
# Remove contamination

## Blobtools visulaizer
### Visualize busco score, read coverage, and taxonomy assignments to remove contaminated contigs.

## Create Blobtools format directory
```
blobtools create --fasta /path/to/curated.fasta a-solani_2
```
## Map nanopore reads to contigs for blobtools
```
minimap2 -ax map-ont \
         -t 40  \
        /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/pilon-bubble-filter-Arthro-only.fasta" \
        /lustre/isaac/scratch/jtorre28/basecalling/A_solani/aulacorthum_sanatized.fasta \
 | samtools sort -@16 -O BAM -o /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/pilon-bubble-filter-Arthro-only.bam -
```
## Add read coverage to blob directory
```
blobtools add --cov /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/pilon-bubble-filter-Arthro-only.bam a-solani_2/
```
## Add BLAST output for taxonomy assignmetn to blobtools blob directory
```
blobtools add --replace --hits /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/pilon-bubble-filter-all-hits  --taxdump ../taxdump/ a-solani_2/
```
## Add Busco output file to blob directory
```
blobtools add --busco artho-only/lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/busco.out/run_hemiptera_odb10/full_table.tsv a-solani_2/
```
## View and download blob directory table and figures
```
blobtools view --local a-solani_2/
```

# Annotation of contigs by e-gapx pipeline.
```
python3 ui/egapx.py /lustre/isaac/scratch/jtorre28/unstable/egapx/examples/input_draft_solani.yaml -e singularity -w tmp-sol -o sol_out
```
## Funtional annotation with EnTAP singularity image 
> Homology searches done with NCBI NR database, uniprot tremble database, uniprot swissprot database, and the ref-seq inverebrate partition database
```
singularity exec entap.sif EnTAP --runN --run-ini entap_run.params --entap-ini entap_config.ini -d /lustre/isaac/scratch/jtorre28/database/swissprot/swissprot.dmnd -d /lustre/isaac/scratch/jtorre28/database/ref-seq/invert/invertebrate-all.protein.dmnd -d /lustre/isaac/scratch/jtorre28/database/cluster-nr/nr.dmnd -d /lustre/isaac/scratch/jtorre28/entap/entap_outfiles_solani1/bin/uniprot_trembl.dmnd
```
#Blobtools 

##Create blobtools directory from fasta file

```
blobtools create --fasta /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/pilon-bubble-filter-Arthro-only.fasta A-solani_final
```

## Add read coverage data to blob directory

```
blobtools add --cov /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/pilon-bubble-filter-Arthro-only.bam A-solani_final
```

## Add Busco score results to blob directory

```
blobtools add --busco /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/artho-only/busco.out/run_hemiptera_odb10/full_table.tsv A-solani_final
```

## Add blast search results of each contig to blob directory

```
blobtools add --replace --hits /lustre/isaac/scratch/jtorre28/foxgloves/purged/purged2/pilon-bubble-filter-all-hits A-solani_final
```



# Figures

### Purge haplotigs read coverage histogram

![aligned2 bam histogram 200](https://github.com/user-attachments/assets/c80542e7-289d-4efd-bbba-4188d907cae5)


### Snail plot of final assembly 

![A-solani_final snail](https://github.com/user-attachments/assets/103547db-8009-4d42-b60c-7d40e056574d)


### Genome scope K-mer histogram using 17-mer counts wiht illumina reads

![plot](https://github.com/user-attachments/assets/36d18029-e657-4d81-b6f2-94693685022f)

> Genome scope length and coverage results:

![image](https://github.com/user-attachments/assets/d1a50dd8-a7dc-442a-b66b-1381a9139359)

>> Genome Scope model and statistics

![image](https://github.com/user-attachments/assets/edfb299a-f4f8-490c-9ccc-f7b44587d8bd)







