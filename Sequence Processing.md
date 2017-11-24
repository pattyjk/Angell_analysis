# Analsis of 16S Illumina sequencing data

All analyses were performed in USEARCH v. 10.0.240 with the UPARSE OTU picking method. Subsequent analyses were performed in QIIME v. 1.8. 

## Merge Paired End Reads
```
#decompress the reads
gunzip *.gz

mkdir mergedfastq

usearch -fastq_mergepairs *R1*.fastq -relabel @ -fastq_maxdiffs 10 -fastqout mergedfastq/merged.fq -fastq_merge_maxee 1.0 -fastq_minmergelen 250 -fastq_maxmergelen -fastq_pctid 80
```

## Dereplicate sequences
```
usearch -derep_fulllength mergedfastq/merged.fq -fastqout mergedfastq/uniques_combined_merged.fastq -sizeout
```

## Remove Singeltons
```
usearch -sortbysize mergedfastq/uniques_combined_merged.fastq -fastqout mergedfastq/nosigs_uniques_combined_merged.fastq -minsize 2
```

## Precluster Sequences
```
usearch -cluster_fast mergedfastq/nosigs_uniques_combined_merged.fastq -centroids_fastq mergedfastq/denoised_nosigs_uniques_combined_merged.fastq -id 0.9 -maxdiffs 5 -abskew 10 -sizein -sizeout -sort size
```

## Reference-based OTU picking against the 13.8 version of GreenGenes
```
usearch -usearch_global mergedfastq/denoised_nosigs_uniques_combined_merged.fastq -id 0.97 -db /mnt/home/kearnspa/gg_13_8_otus/rep_set/97_otus.fasta  -strand plus -uc mergedfastq/ref_seqs.uc -dbmatched mergedfastq/closed_reference.fasta -notmatchedfq mergedfastq/failed_closed.fq
```

## Sort by size and then de novo OTU picking on sequences that failed to hit GreenGenes
```
usearch -sortbysize mergedfastq/failed_closed.fq -fastaout mergedfastq/sorted_failed_closed.fq

usearch -cluster_otus mergedfastq/sorted_failed_closed.fq -minsize 2 -otus mergedfastq/denovo_otus.fasta -relabel OTU_dn_ -uparseout denovo_out.up
```

## Combine the rep sets between de novo and reference-based OTU picking
```
cat mergedfastq/closed_reference.fasta mergedfastq/denovo_otus.fasta > mergedfastq/full_rep_set.fna
```

## Map rep_set back to pre-dereplicated sequences and make OTU tables
```
usearch -usearch_global mergedfastq/merged.fq -db mergedfastq/full_rep_set.fna  -strand plus -id 0.97 -uc OTU_map.uc -otutabout OTU_table.txt -biomout OTU_jsn.biom
```





# Switch to QIIME

## Assign taxonomy to GreenGenes v 13.8
```
assign_taxonomy.py -i full_rep_set.fna -o taxonomy -r /mnt/home/kearnspa/gg_13_8_otus/rep_set/97_otus.fasta -t /mnt/home/kearnspa/gg_13_8_otus/taxonomy/97_otu_taxonomy.txt
```

## Add taxonomy to OTU table
```
biom add-metadata -i OTU_jsn.biom -o otu_table_tax.biom --observation-metadata-fp=taxonomy/full_rep_set_tax_assignments.txt --sc-separated=taxonomy --observation-header=OTUID,taxonomy
```

## Filter non-bacteria/archaea
```
filter_taxa_from_otu_table.py -i otu_table_tax.biom -o otu_table_tax_filt.biom -n o__Streptophyta,o__Chlorophyta,f__mitochondria,Unassigned
```

## Align sequences to GreenGenes with PyNast
```
align_seqs.py -i full_rep_set.fna -o alignment -t /mnt/home/kearnspa/gg_13_8_otus/rep_set_aligned/97_otus.fasta
```

## Filter excess gaps from alignment
```
filter_alignment.py -i alignment/full_rep_set_aligned.fasta -o alignment/filtered_alignment
```

## Make phylogeny with fasttree
```
make_phylogeny.py -i alignment/filtered_alignment/full_rep_set_aligned_pfiltered.fasta -o rep_set.tre
```

## Summarize the OTU table and rarefy OTU table to lowest sequencing depth
```
biom summarize-table -i otu_table_tax_filt.biom -o otu_table_summary.txt

single_rarefaction.py -d 27000 -o single_rare.biom -i otu_table_tax_filt.biom
```

## Calculate alpha and beta diversity
```
beta_diversity.py -m bray_curtis,unweighted_unifrac,weighted_unifrac -i otu_table_tax_filt.biom -o beta_div -t rep_set.tre

alpha_diversity.py -m PD_whole_tree,shannon -i single_rare.biom -o alpha -t rep_set.tre
```
