#All analyses were performed in USEARCH v. 10.0.240 with the UPARSE OTU picking method. Subsequent analyses were performed in QIIME v. 1.8. 

##Merge Paired End Reads

```
#decompress the reads
gunzip *.gz

mkdir mergedfastq

usearch -fastq_mergepairs *R1*.fastq -relabel @ -fastq_maxdiffs 10 -fastqout mergedfastq/merged.fq -fastq_merge_maxee 1.0 -fastq_minmergelen 250 -fastq_maxmergelen -fastq_pctid 80
```
