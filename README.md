# NGA-LTER-Qiime2
#This Qiime2 Pipeline was used to process demultiplexed 16S/18S Amplicon sequences from the NGA-LTER
#########################################################################################################################################################################################
#First, run the Qiime2 pipeline on your demutiplexed sequences. 

#Activate Qiime2.2021.2 in Chinook
conda activate qiime2.2021.2

#Import demulitplexed paired-end sequences into Qiime2
qiime tools import --type 'SampleData[PairedEndSequencesWithQuality]' --input-path [Your Manifest File] --output-path combined-demux.qza --input-format PairedEndFastqManifestPhred33V2

#View your demultiplexed file stats - this will tell you 
qiime demux summarize --i-data [Your demultiplexed data file] --o-visualization demux.qzv

#Quality filtering step with dada2- this step will remove chimeras and singletons and correct errors in the sequences.
#The output directory is where you will perform the rest of the Qiime pipeline and analyze your data. 
qiime dada2 denoise-paired --i-demultiplexed-seqs [Your demultiplexed data file] --p-trunc-len-f 0 --p-trunc-len-r 0 --p-n-threads 20 --output-dir filtered

#Cluster your sequences based on identity- here we will use 100% identity for ASVs. This will create an output directory in which you will classify your sequences. 
qiime vsearch cluster-features-de-novo --i-table table.qza --i-sequences representative_sequences.qza --p-perc-identity 1.00 --p-threads 50 --output-dir ASVs

#Classify your sequences with a classifier of choice trained for Qiime2. 
qiime feature-classifier classifiy-sklearn --i-reads clustered_sequences.qza --i-classifier [Your chosen classifier] --p-n-jobs 20 --output-dir classify

#Align your metadata file with your original table- this will allow you to visualize your data
qiime feature-table filter-samples --i-table table.qza --m-metadata-file [Your metadata file] --o-filtered-table filtered_table.qza

#Visualize the taxonomy of your samples. This code will create a barplot which can be viewed in Qiime2view upon export. Congratulations!
qiime taxa barplot --i-table filtered_table.qza --i-taxonomy classification.qza --m-metadata-file [Your metadata file] --o-visualization visualization.qzv
#########################################################################################################################################################################################
#Filtering your classified sequences. You may want to filter your table based on a variety of factors. The following code can be used to do so. 

#Frequency-based filtering. This step will remove samples containing a sequence number below a certain threshold. 
qiime feature-table filter-samples --i-table [Your table] --p-min-frequency 1500 --o-filtered-table sample-frequency-filtered-table.qza

#Sample-based filtering. This step will remove samples which you do not want to include in your final analysis. You will need to make a second metadata file that only includes the 
#samples you want. 
qiime feature-table filter-samples --i-table [Your table] --m-metadata-file [Your second metadata file] --o-filtered-table id-filtered-table.qza

#Sequence-based filtering. This step will remove unwanted sequences from your table- unassigned, chloroplast, vertebrate, etc. You can also determine what sequences you'd like to keep
#It is not necessary to use both "--p-include" and "--p-exclude" in your code but you can. 
qiime taxa filter-table --i-table [Your table] --i-taxonomy classification.qza --p-include [Samples to include] --p-exclude [Samples to be excluded] 
--o-filtered-sequences sequence-filtered-table.qza
#########################################################################################################################################################################################
#Statistics and visualization. Qiime2 can perform a number of statistical tests and visualizations of said statisitcs.
#Before running these statistics, you need to make build a phytogenetic tree by using Mafft to run a multiple sequence alignment. This step should be performed in your "filtered" 
#directory 

#Run the multiple sequence alignment 
qiime alignment mafft --i-sequences representative_sequences.qza --o-alignment aligned-rep-seqs.qza

#Filter highly variable sequences
qiime alignment mask --i-alignment aligned-rep-seqs.qza --o-masked-alignment masked-aligned-rep-seqs.qza

#Create an unrooted tree
qiime phylogeny fasttree --i-alignment masked-aligned-rep-seqs.qza --o-tree unrooted-tree.qza

#Root your tree based on the longest root
qiime phylogeny midpoint-root --i-tree unrooted-tree.qza --o-rooted-tree rooted-tree.qza

#Create a base statistics directory with the table of your choice
qiime diversity core-metrics-phylogenetic --i-phylogeny rooted-tree.qza --i-table [Your chosen table] --p-sampling-depth [Your choice of sampling depth] 
--m-metadata-file [Your metadata file] --output-dir core-metrics-results

#You now have many statisitcal tests of your data within the core-metrics-results folders. You can now make visualizations of this data for Qiime2view. The following code will give you 
#visualization for Shannon Diversity
qiime diversity alpha-group-significance --i-alpha-diversity shannon_vector.qza --m-metadata-file [Your metadata file] --o-visualization shannon_group-significance.qzv

#You can also make a v0 graph of beta diversity based on several metrices for Qiime2view. You can look for differences based on a chosen categorical variable. 
qiime diversity beta-group-significance --i-distance-matrix [Chosen beta diversity method] --m-metadata-file [Your metadata file] --m-metadata-column [Chosen metadata variable] 
--o-visualization unweighted-unifrac-month-significance.qzv --p-pairwise
