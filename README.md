# cellatac

Sanger Cellular Genetics ATAC-seq pipeline by Luz Garcia Alonso, Ni Huang and Stijn van Dongen.

**cellatac** takes scATAC-seq aligned data (such as the fragments file from Cell Ranger ATAC) and outputs a _count matrix of accessible chromatin peaks by cell_ (i.e. analogous to the `filtered_peak_bc_matrix` from Cell Ranger ATAC). The output matrix can then be used for dowstream analysis in Seurat, Scanpy, cisTopic or any other tool.

There are two relevant steps in the pipeline:
 - **Multiplets identification (optional)**. Cell Ranger ATAC 1.2 identifies [multiplets](https://www.nature.com/articles/s41467-020-14667-5), selects the "dominant" barcode in the multiplet and removes the rest. We think all barcodes from a multiplet should be aggregated to avoid losing data. **cellatac** uses an alternative approach to identify these multiplets and aggregates them. Code courtesy of Ni Huang.
- **Accessible chromatin peak calling**. Cell Ranger ATAC identifies the peaks by aggregating the signal of all the barcodes in the sample. There are some papers reporting that this may be unsuitable to detect peaks appearing in rare cell types/states. **cellatac** uses [Cusanovich approach](https://www.sciencedirect.com/science/article/pii/S0092867418308559) to increase the peak detection sensitivity by, first, identifying cell clusters on a windows x cell rather than peaks per cell matrix, and then doing a peak calling for each cluster. 

### Basic workflow

0. **Identify and merge multiplets** (optional). 
1. **Compute window coverage**. The genome is broken into 5kb windows and then each cell is scored for insertions in each window, generating a binary matrix (large and sparce) of windows by cells. Note that if multiple samples are provided, these are aggregated into a unique matrix.
2. **Cluster cells based on window coverage**. Matrix is filtered to retain only top 50K most commonly used windows. Using `Signac`, the binary matrix is normalized with Term Frequency-Inverse Document Frequency (TF-IDF) approach followed by a dimensionality reduction step using Singular Value Decomposition (SVD). The first LSI component is ignored as it often captures sequencing depth (technical variation) rather than biological variation. The 2-30 top remaining components are used to perform graph-based Louvain clustering (at a X resolution) and clusters are reported.
3. **Accessible chromatin peak calling per cluster**. Peaks are called separately on each cluster using `macs2`.
4. **Merge per-cluster peaks and generate peak by cell matrix.** Peaks from all clusters are merged into a master peak set (i.e. overlapping peaks are aggregated), and the corresponfding peak by cell matrix (indicating any reads occuring in each peak for each cell) is reported. Note that if multiple samples are provided, these are aggregated into a unique matrix. **This is the relevant matrix that you should use for clustering.**

### Cellatac currently has/implements/supports

* Optional merging of multiplets.
* The clustering approach from the Cusanovich 2018 manuscript.
* A clustering step utilising Seurat.
* User-specified clustering.
* Peak/cell matrix based on merging per-cluster peaks.
* Peak/cell matrix per-cluster.


## Running cellatac

### Cellatac needs

* Singularity


### Useful options

```
--mermul true           merge multiplets using CR bam file
--mermul false          [default] use CR fragments.tsv.gz

--usecls __seurat__        [default] use Seurat/Signac approach resembling Cusanovich. It uses Louvain clustering instead.
--usecls __cusanovich__    use cusanovich-strict approach. It uses bi-clustering of cells and windows based on cosine distances using the ward algorithm.
--usecls <filename>        use custom clustering

--mergepeaks true       [default] merge cluster peaks, compute master cell/peak matrix
--perclusterpeaks false [default] computer per-cluster cell/peak matrix  
                            Note both can be set to true.

--cellbatchsize 500     [default] parallelisation bucket size (number of cells per bucket)
--nclades 10            [default] number of clusters to use (only applies to cusanovich-strict approach)
--sampleid <tag>        use <tag> in naming outputs. Not yet consistently applied
```


### Example invocations

This pipeline will need a singularity installation.  It supports two executing
platforms, *local* (simply execute on the machine you're currently on) and
*lsf*. To use the latter specify `-profile lsf`.


```
source=cellgeni/cellatac

manifest=/some/path/to/singlecell.csv
posbam=/some/path/to/possorted_bam.bam
fragments=/some/path/to/fragments.tsv.gz

cellbatchsize=400
nclades=10

nextflow run $source        \
  --cellcsv $manifest       \
  --fragments $fragments    \
  --cellbatchsize $cellbatchsize   \
  --posbam $posbam          \
  --outdir results          \
  --sampleid CR12345678     \
  -profile local            \
  --mermul true             \
  --usecls __seurat__       \
  --mergepeaks true         \
  -with-report reports/report.html \
  -resume -w work -ansi-log false \
  -config my.config
```

where `my.config` supplies singularity mount options and tells nextflow how many CPUs it can utilise
when using the local executor, e.g.

```
singularity {
  runOptions = '-B /some/path1 -B /another/path2'
	cacheDir = '/home/jovyan/singularity/'
}

executor {
    cpus   = 56
    memory = 600.GB
}
```

To run multiple samples:

```
nextflow run $source        \
  --muxfile mux.txt         \
  --cellbatchsize $cellbatchsize   \
  --outdir results          \
  -profile local            \
  --usecls __seurat__       \
  --mermul false            \
  --mergepeaks true         \
  -with-report reports/report.html \
  -resume -w work -ansi-log false \
  -c my.config
```

where `mux.txt` is a tab separated file that looks like this:

```
1   sampleX   /path/to/cellranger/output/for/sampleX
2   sampleY   /path/to/cellranger/output/for/sampleY
3   sampleZ   /path/to/cellranger/output/for/sampleZ
4   sampleU   /path/to/cellranger/output/for/sampleU
```

The first column will be used to make the barcodes in each sample unique across the merged samples. As
such it can be anything, but it is suggested to simply use a range of integers starting at 1, or to
use the last one or two signficant digits of the sample ID provided they are unique to each sample.

The cellranger output directories need not contain the full output. Currently the pipeline expects
these files:

```
fragments.tsv.gz  possorted_bam.bam singlecell.csv
```

When running multiple samples, the bam file is only used for its header. It is possible to
substitute the original bam file with the output of `samtools view -H possorted_bam.bam`. This can
be useful if it is necessary to copy the data prior to running this pipeline; it is not necessary
in this case to copy the full position sorted bam file (they tend to be very large).
Currently it is necessary that the substituted file has the same name `possorted_bam.bam`.


## Outputs



