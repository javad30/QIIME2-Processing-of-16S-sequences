# QIIME2-Processing-of-16S-sequences
Evidence for deterministic processes that shape teleost microbiome
Qiime 2

##Import the fastq file into qiime

qiime tools import --type MultiplexedSingleEndBarcodeInSequence --input-path muxed-se-barcode-in-seq/sequences.fastq.gz --output-path multiplexed-seqs.qza

#Demultiplexing the data 
qiime cutadapt demux-single  --i-seqs multiplexed-seqs.qza --m-barcodes-file metadata.tsv --m-barcodes-column Barcode  --p-error-rate 0  --o-per-sample-sequences demultiplexed-seqs.qza --o-untrimmed-sequences untrimmed.qza --verbose

#Remove the primer from the sequence file 
qiime cutadapt trim-single --i-demultiplexed-sequences demultiplexed-seqs.qza --p-front ACATTCTGGGATGACAGGGGTA --p-error-rate 0 --o-trimmed-sequences trimmed-seqs.qza --verbose

#Visualtion of the sequence file and qulity checking 
qiime demux summarize --i-data trimmed-seqs.qza --o-visualization trimmed-seqs.qzv 

Amplicon Sequence Variant (ASV) picking and chimera removing 
qiime dada2 denoise-pyro --i-demultiplexed-seqs trimmed-seqs.qza --p-trim-left 0 --p-trunc-len 280 --p-n-threads 20 --o-representative-sequences rep-seqs-dada2.qza --o-table table-dada2.qza  --o-denoising-stats stats-dada2.qza
## --p-trim-left 20 --p-trunc-len 280  are selected based on Trimmed-seqs.qzv visualation 
##--p-n-threads 32 (is selected based on the computer capabilities)

##Checking ead per samples and number of sequence before and after removing chimers  
qiime metadata tabulate --m-input-file stats-dada2.qza --o-visualization stats-dada2.qzv

##Phylogeny
qiime phylogeny align-to-tree-mafft-fasttree  --i-sequences rep-seqs.qza  --o-alignment aligned-rep-seqs.qza  --o-masked-alignment masked-aligned-rep-seqs.qza  --o-tree unrooted-tree.qza  --o-rooted-tree rooted-tree.qza




@@IF you have two or more seqence file and if you want to combine them then you need to follow these steps (https://docs.qiime2.org/2020.6/tutorials/fmt/):

qiime feature-table merge  --i-tables table-1.qza  --i-tables table-2.qza  --o-merged-table table.qza

qiime feature-table merge-seqs  --i-data rep-seqs-1.qza  --i-data rep-seqs-2.qza  --o-merged-data rep-seqs.qza

qiime feature-table summarize  --i-table table.qza  --o-visualization table.qzv  --m-sample-metadata-file sample-metadata.tsv

qiime feature-table tabulate-seqs  --i-data rep-seqs.qza  --o-visualization rep-seqs.qzv

qiime phylogeny align-to-tree-mafft-fasttree  --i-sequences rep-seqs.qza  --o-alignment aligned-rep-seqs.qza  --o-masked-alignment masked-aligned-rep-seqs.qza  --o-tree unrooted-tree.qza  --o-rooted-tree rooted-tree.qza

##Alpha and beta diversity (this command can also be run separately)
qiime diversity core-metrics-phylogenetic --i-phylogeny rooted-tree.qza  --i-table table.qza  --p-sampling-depth 3100  --m-metadata-file sample-metadata.tsv  --output-dir core-metrics-results
 
 #--p-sampling-depth  is selected based on your minimum reads per your samples or rarefaction plot 
 
 
 ##Add taxonomy (https://docs.qiime2.org/2020.6/tutorials/feature-classifier/)


Download the SILVA reference file (gg-13-8-99-nb)

qiime feature-classifier fit-classifier-naive-bayes   --i-reference-reads ref-seqs.qza   --i-reference-taxonomy ref-taxonomy.qza   --o-classifier classifier.qza

qiime feature-classifier classify-sklearn --i-classifier classifier.qza  --i-reads rep-seqs.qza  --o-classification taxonomy.qza

qiime metadata tabulate  --m-input-file taxonomy.qza  --o-visualization taxonomy.qzv

##Add taxa information to ASV table 

qiime tools export --input-path table.qza --output-path exported

qiime tools export --input-path taxonomy.qza --output-path exported

cp exported/taxonomy.tsv biom-taxonomy.tsv

##Open the biom-taxonomy.tsv and change the first line of biom-taxonomy.tsv (i.e. the header) to this:

#OTUID	taxonomy	confidence

biom add-metadata -i exported/feature-table.biom -o table-with-taxonomy.biom --observation-metadata-fp biom-taxonomy.tsv --sc-separated taxonomy

biom convert -i table-with-taxonomy.biom -o table.from_biom_w_taxonomy.txt --to-tsv --header-key taxonomy


##Useful for filtering (after alpha diversity)

##Deleting mitocondri, chloroplast,Unassigned,d__Eukaryot

qiime taxa filter-table  --i-table table.qza  --i-taxonomy taxonomy.qza  --p-exclude mitochondria,chloroplast,Unassigned,d__Eukaryota  --o-filtered-table table-no-mitochondria_Eukaryota-chloroplast.qza

##Filtering samples less than 3000 reads

qiime feature-table filter-samples  --i-table table-no-mitochondria_Eukaryota-chloroplast.qza --p-min-frequency 3000  --o-filtered-table sample-frequency-filtered-table.qza


qiime feature-table filter-features  --i-table sample-frequency-filtered-table.qza  --p-min-samples 2  --o-filtered-table sample-contingency-filtered-table.qza

qiime feature-table filter-samples  --i-table second_3000_sample-frequency-filtered-table.qza  --p-min-features 10  --o-filtered-table feature-contingency-filtered-table.qza


biom convert -i table-with-taxonomy.biom -o table.from_biom_w_taxonomy.txt --to-tsv --header-key taxonomy

##ANCOM

ANCOM operates on a FeatureTable[Composition] QIIME 2 artifact, which is based on frequencies of features on a per-sample basis, but cannot tolerate frequencies of zero. To build the composition artifact, a FeatureTable[Frequency] artifact must be provided to add-pseudocount (an imputation method), which will produce the FeatureTable[Composition] artifact.

qiime composition add-pseudocount  --i-table table.qza  --o-composition-table table1.qza


qiime composition ancom --i-table table1.qza --m-metadata-file sample-metadata.tsv  --o-visualization ANCOM.qza 








