# PEQG_2026_graph_QC

This repo contains a test data set and the info needed to run basic exploration and QC steps on a collection of sequences and a corresponding chromosome-level graph file. The total size of input files is ~1.7 GB, which includes a ~1.2-GB index file typically produced in Step 3 below. This index can be downloaded, used for Step 3, and deleted when moving to Step 4. Runtime estimates calculated on a MacBook Pro with an M2 chip and 16 GB RAM.

1. All programs needed can be installed in a Conda environment. The programs we'll be using are [PanKmer](https://salk-tm.gitlab.io/pankmer/) and [panacus](https://github.com/codialab/panacus). Create a new environment and install the programs + dependencies:

	```
	conda create -c conda-forge -c bioconda -n pan_QC \
	  python=3.10 cython gff2bed more-itertools pybedtools \
	  python-newick pyfaidx rust seaborn upsetplot urllib3 \
	  tabix dash-bootstrap-components panacus
  
	conda activate pan_QC

	pip install pankmer
	```

2. Download the `input_data` directory of this repo, put it wherever you'd like, and navigate to it in your terminal app. Also download the [graph GFA file](https://drive.google.com/file/d/1EJnVK4mJXdjzWpcRyuppC2NlP6b5Xw2k/view?usp=sharing) and the [PanKmer index file](https://drive.google.com/file/d/1OrXsr4iX-X_u6MVUOl2p3VdPQYHQFofb/view?usp=sharing) stored on Google Drive and add them to the `input_data` directory.

3. **PanKmer**. First we'll take a look at the patterns of diversity represented in our collection of sequences. These sequences come from Chr04K of switchgrass (*Panicum virgatum*)--a polyploid species of bioenergy interest--and comprise 20 haplotypes from 10 individuals, spanning combinations of 3 subpopulations and 3 ecotypes[^1]. PanKmer is a really quick way to get a snapshot of sequence divergences across a collection of FASTA files.

   From inside `/input_data/`, make an output directory and create the *k*-mer index of the input sequences (runtime ~1 hour; just use the downloaded version for now):
	```
	mkdir ../pankmer_output
	
	pankmer index \
		-g switchgrass_n20-Chr04K \
		-o switchgrass_n20-Chr04K_index.tar
	```

   Next, we'll create an adjacency matrix from the *k*-mer index and plot Jaccard similarities as a heatmap:  
   ```
	pankmer adj-matrix \
		-i switchgrass_n20-Chr04K_index.tar \
		-o ../pankmer_output/switchgrass_n20-Chr04K_adj_matrix.tsv

	pankmer clustermap \
		--metric jaccard \
		--width 15 \
		--height 15 \
		-i ../pankmer_output/switchgrass_n20-Chr04K_adj_matrix.tsv \
		-o ../pankmer_output/switchgrass_n20-Chr04K_adj_matrix_jaccard.pdf
   	```

   From the heatmap, some pretty clear patterns emerge: we see three clear clusters of haplotypes and strong divergence among those clusters, which correspond to the three subpopulations mentioned in the workshop. We even see quite a bit of divergence between haplotypes within the same individual genotype; there's a lot of sequence diversity here. Given the strong divergence among the subpopulations, selecting a haplotype from any of the three will almost guarantee at least a slight bias towards that subpopulation's sequences in the graph structure over the other two subpopulations.

	<details>
  	<summary>
		Click to view heatmap
  	</summary>
  
	![pankmer_heatmap](https://github.com/avril-m-harder/PEQG_2026_graph_QC/blob/main/expected_output/switchgrass_n20-Chr04K_adj_matrix_jaccard.jpg)

	</details>
	
4. **panacus**. Based on the PanKmer assessment, let's say we've gone ahead with construction of a graph with Minigraph-Cactus, with Gulf_A_HAP1 selected as the primary reference sequence, or the backbone of the graph. Typically, assessments of a graph are done on a whole-genome level, but we'll use a single chromosome GFA file as an example.
   
   Panacus is a great program for getting a sense of the diversity represented in a graph. It requires a config file, where you can define the analyses you'd like to run. For example, the below config file will direct Panacus to:
   
	<details>
	<summary>
		Config example:
	</summary>
   
	```
	- graph: ./switchgrass_n20-Chr04K.gfa.gz
	  grouping: Sample
	  analyses:
		- !Info
		- !Hist
	  	  count_type: All
		- !Growth
	      coverage: 1,2,2,2
	      quorum: 0,0.1,0.5,1
	  	  count_type: All
		- !OrderedGrowth
	  	  count_type: Bp
	  	  order: ./hist_order.txt
	  	  coverage: 1,2,2,2
	  	  quorum: 0,0.1,0.5,1
	```
   
	</details>

   * Analyze the GFA file for Chr04K, printing some basic stats (!Info) grouped by sample name (can be set to "Haplotype" to specifically analyze different haplotypes from the same sample by using [PanSN](https://github.com/pangenome/PanSN-spec) haplotype naming convention) 
   * !Hist: Create a histogram describing patterns of shared information (i.e., bases, edges, and nodes) among haplotypes in the graph
   * !Growth: Calculate an exact growth curve based on those same patterns, highlighting the number of bases, edges, and nodes observed in various sequence-sharing categories as defined by the `coverage` and `quorum` specifications (i.e., unique, shared by ≥ 10% of haplotypes, shared by ≥50% of haplotypes, and core)
   * !OrderedGrowth: Calculate a growth curve for the graph by analyzing haplotypes in the specified order (memory intensive, so must specify how to summarize; here, 'Bp')

   Once the config file is ready, we can create an output directory and generate a report containing the requested results with a single command:
	```
	mkdir ../panacus_output
	
	panacus report panacus_report.yaml > \
	../panacus_output/panacus-switchgrass_n20-Chr04K.html
	```
   
   The output will all be contained in a single HTML file from which you can export tables for custom plotting or SVGs for figure editing. Open this file locally to explore your results. (You can also download this [example report](https://github.com/avril-m-harder/PEQG_2026_graph_QC/blob/main/expected_output/panacus-switchgrass_n20-Chr04K.html) to explore and compare your results against.)
   
   Some take-homes from the report:
   * Coverage Histogram: Each plot shows roughly the same trend, which is an overrepresentation of unique sequence and very little core sequence. (It is worth comparing the base and node plots and keeping in mind that nodes can be anywhere from 1 to 1,024 bases in length.)
   * Pangenome Growth: The relative amounts of core and unique sequences are highlighted again here. Also note that the curve does not appear to be reaching much of an asymptote. To directly estimate openness, you can download the table from the HTML report and apply  an additional (deprecated) command to it, e.g.:
	   ```
	   panacus-visualize \
		-f pdf \
		-e \
		../expected_output/pan-growth-switchgrass-n20-chr04k.gfa.gz--group-by-sample-growth-bp__Placeholder\ Filename__table.tsv > \
		../panacus_output/growth_out-w_equations.pdf
	   ```
   The plots written by this command include growth curve equations and estimates for $\alpha$. Generally curves where $\alpha$ < 1 can be interpreted as open, whereas $\alpha$ > 1 indicates a more closed collection of sequences.
   * Pangenome Info: General graph summary information that can be used to calculate a compression index (i.e., total length of a graph/total length of sequences input into a graph builder), assess haplotype-specific clipping, etc.
   * Ordered Growth: Unlike 'Pangenome Growth,' which is an exact growth curve, this plot shows how the graph grows with haplotypes are added in a specified order. Notice the extra little jumps in y-values when going from one subpopulation to the next.
   
[^1]: [Lovell et al. 2021, *Nature*](https://doi.org/10.1038/s41586-020-03127-1)