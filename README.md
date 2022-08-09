# Workshop for VEME 2022

[![Open with Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/blab/veme-2022/HEAD)
[![Open with Gitpod](https://img.shields.io/badge/Open%20with-Gitpod-908a85?logo=gitpod)](https://gitpod.io/#https://github.com/blab/veme-2022.git)

## Prerequisites

  - [Install Nextstrain]() on your computer or launch one of the virtual machine providers listed above.

## Scope

This workshop focuses on specific file formats, command line instructions required to analyze a specific pathogen, and instructions to visualize and interpret that pathogen's phylogeny in Auspice as a way of exposing the learners to the details of Nextstrain's genomic epidemiology tools.
This workshop does not include discussions about installation of tools, data curation, or workflow design; these topics are critical but outside the scope of a short workshop on genomic epidemiology.

## Learning objectives

By the end of this workshop, attendees will know how to:

  - identify the necessary input files to perform a genomic epidemiology analysis with Nextstrain
  - run commands in Nextstrain’s bioinformatics toolkit including Augur, Nextalign, and Nextclade to convert input genomes and metadata into an annotated phylogenetic time tree that can be visualized locally or online
  - inspect and understand the contents of Nextstrain toolkit command outputs
  - visualize and interpret a phylogenetic tree produced by Nextstrain’s bioinformatics toolkit using Auspice or [auspice.us](http://auspice.us)

## Introduction

### Why do we need real-time genomic epidemiology?

At the beginning of the SARS-CoV-2 pandemic, we knew little about this specific new virus, but we did have previous knowledge from SARS-CoV-1 and MERS-CoV outbreaks and a handful of complete SARS-CoV-2 genomes and metadata.
Together, these data and previous knowledge allowed the Nextstrain team to prepare [a situation report documenting the state of the pandemic at its earliest stages](https://nextstrain.org/narratives/ncov/sit-rep/2020-01-23).
[Genomic epidemiology enabled detection of community spread in the Seattle area early in the pandemic](https://nextstrain.org/narratives/ncov/sit-rep/2020-03-05?n=10) before anyone was aware that SARS-CoV-2 was already so widespread in the community.
These tools have been a key component of New Zealand's pandemic response, as documented in [a Nextstrain narrative describing specific outbreaks in New Zealand](https://nextstrain.org/community/narratives/ESR-NZ/GenomicsNarrativeSARSCoV2/aotearoa-border-incursions).

To produce "real-time" analyses, Nextstrain relies on maximum likelihood methods.
Even if you plan to use Bayesian methods, Nextstrain's tools enable rapid prototyping of subsampled datasets to use in subsequent Bayesian analyses.

### How do we use Nextstrain to perform genomic epidemiology analyses?

The process for creating a Nextstrain analysis generally requires the following steps.

  - Curate data.
    - Collect genome sequences.
    - Collect genome metadata.
    - Prepare pathogen-specific configuration files and parameters (e.g., reference genomes, gene coordinates, clock rates, etc.).
  - Analyze data.
    - Run commands manually or in a workflow.
    - Run commands locally or in the cloud.
    - We will focus on how to run commands and understand their inputs and outputs.
  - Visualize and interpret analysis outputs.
    - Run a local Auspice server.
    - Drag-and-drop onto [auspice.us](http://auspice.us).
    - Upload data to GitHub or Nextstrain Groups and view through [nextstrain.org](http://nextstrain.org).

## Build an annotated time-scaled phylogeny of SARS-CoV-2

Inspect input files for SARS-CoV-2 analysis.
We will use a curated reference genome and annotations from Nextclade.
Inspect the contents of the reference genome FASTA and the gene map with genomic coordinates in [GFF format](https://github.com/The-Sequence-Ontology/Specifications/blob/master/gff3.md).

``` bash
head data/reference.fasta
head data/genemap.gff
```

Next, inspect the genome sequences and metadata we have curated for this analysis.
These consist of two text files, one in [FASTA format](https://www.ncbi.nlm.nih.gov/genbank/fastaformat/) and the other in a [tab-separated values (TSV) format](https://www.loc.gov/preservation/digital/formats/fdd/fdd000533.shtml).
Genome sequences have:

  - One unique name per genome sequence that matches the name in the metadata.
  - One FASTA sequence per genome.

``` bash
head data/sequences.fasta
```

Note that Nextstrain also supports VCF files, as an alternate representation of sequences.

Genome metadata have:

  - One tab-delimited record per genome sequence with a “strain” name that matches the genome sequence name.
  - Required columns including “strain” and “date”.
  - As many additional columns as you like.

``` bash
head data/metadata.tsv
```

To understand the evolutionary and epidemiological history of these samples, we need to:

  - select a representative set of high-quality samples
  - align their genomes
  - infer a phylogeny
  - infer a time-scaled phylogeny
  - infer ancestral sequences and traits
  - visualize the annotated phylogeny

We start by selecting the representative set of high-quality samples.
We determine the quality of the original data based on attributes of both the genome sequences and metadata.
First, we calculate statistics across all genome sequences, to identify potentially problematic sequences to filter from the analysis.
We will use [Augur](https://docs.nextstrain.org/projects/augur/en/stable/index.html) to calculate these statistics and store them in a results directory.

Create the results directory.

``` bash
mkdir results/
```

Look at available Augur subcommands.

``` bash
augur -h
```

Look at the help text for a specific Augur subcommand.

``` bash
augur index -h
```

Index genome sequences, calculating statistics across each genome.

``` bash
augur index \
  --sequences data/sequences.fasta \
  --output results/sequence_index.tsv
```

Inspect the sequence index.

``` bash
head results/sequence_index.tsv
```

Filter the data to eliminate low-quality or undesired data based on genome sequence or metadata attributes.
In the following command, we filter by date, sequence length, a query on the metadata attributes, and an explicit list of strains to exclude.
We also force the inclusion of reference records that we will need for rooting the tree later.

``` bash
augur filter \
  --metadata data/metadata.tsv \
  --sequences data/sequences.fasta \
  --sequence-index results/sequence_index.tsv \
  --include config/include.txt \
  --min-date 2021-01-01 \
  --min-length 27000 \
  --output-metadata results/filtered_metadata.tsv \
  --output-sequences results/filtered_sequences.fasta \
  --output-log results/filtered_log.tsv
```

The output from the command shows the total number of samples that were filtered out and force-included.
We can inspect the output log to see why specific samples were filtered or included.

``` bash
head results/filtered_log.tsv
```

After filtering for high-quality data, we often still have more samples than we can reasonably use to infer a phylogeny and we need to subsample our data.
Effective subsampling is a research topic of its own, but most commonly we try to sample evenly through time and space.
This approach attempts to account for sampling bias.
The following command uses Augur's filter subcommand again, this time to select at most 30 samples evenly across all countries and year/month combinations in the metadata.
We also force-include the reference sequence required to root the tree later on.

``` bash
augur filter \
  --metadata results/filtered_metadata.tsv \
  --sequences results/filtered_sequences.fasta \
  --group-by country month \
  --subsample-max-sequences 30 \
  --include config/include.txt \
  --output-metadata results/subsampled_metadata.tsv \
  --output-sequences results/subsampled_sequences.fasta
```

Next, we align the genome sequences of our subsampled data to a single reference genome.
This alignment ensures that all genomes have the same coordinates during tree inference.

> Note: Although we use a reference-based alignment with Nextalign here, you may want to perform a multiple sequence alignment with augur align, depending on your pathogen.
>
> ```bash
> augur align \
>   --sequences results/subsampled_sequences.fasta \
>   --nthreads 1 \
>   --reference-sequence data/reference.fasta \
>   --fill-gaps \
>   --output results/aligned.fasta
> ```

Nextalign produces both an alignment of the nucleotide sequences and amino acid alignments for all genes defined in the given gene map.
Insertions relative to the reference genome appear in an optional comma-separated values (CSV) output.

``` bash
nextalign run \
  --input-ref data/reference.fasta \
  --input-gene-map data/genemap.gff \
  --output-fasta results/aligned.fasta \
  --output-translations "results/aligned.translations.{gene}.fasta" \
  --output-insertions results/aligned.insertions.csv \
  --output-errors results/aligned.errors.csv \
  results/subsampled_sequences.fasta
```

Infer a divergence tree from the alignment.
Augur's tree subcommand is a lightweight wrapper around existing tree builders, providing some standardization of the input alignment and output across tools.
We use IQ-TREE by default, but other options include FastTree and RAxML.

``` bash
augur tree \
  --alignment results/aligned.fasta \
  --output results/tree_raw.nwk
```

We can view this tree and its metadata in [auspice.us](https://auspice.us/) or FigTree.

> Note: All tree builders used by Augur are maximum-likelihood (ML) tools, enabling the “real-time” part of Nextstrain’s mission at the expense of the posterior and more sophisticated models available through Bayesian methods.
> The ML approach enables rapid prototyping to identify genomes to include in a more complex, longer-running Bayesian analysis.

With the alignment, the divergence tree, and the dates per sample from the metadata, we can infer a time-scaled phylogeny with estimated dates for internal nodes of the tree.
Augur's refine subcommand is a lightweight wrapper around [TreeTime](https://github.com/neherlab/treetime).
The following command roots the tree with a specific reference genome that we force-included earlier.

``` bash
augur refine \
  --alignment results/aligned.fasta \
  --tree results/tree_raw.nwk \
  --metadata results/subsampled_metadata.tsv \
  --timetree \
  --root "Wuhan-Hu-1/2019" \
  --output-tree results/tree.nwk \
  --output-node-data results/branch_lengths.json
```

This is the first part of the workflow that produces a "node data JSON" output file.
We will see more of these in subsequent steps.
The node data JSON file is a Nextstrain-specific file standard that stores key/value attributes per node in the phylogenetic tree like clock-scale branch lengths, inferred collection dates, etc.
Unlike the divergence tree builders, `augur refine` names internal nodes (e.g., NODE_0000000) so we can reference them in other downstream tools.

``` bash
less results/branch_lengths.json
```

We now have enough information to export the initial time tree and its metadata for visualization in Auspice.
This export step requires at least a Newick tree and a node data JSON file to produce an "Auspice JSON" file, another Nextstrain-specific file standard that represents a tree, its metadata, its node data, and details about how these data should all be visualized in Auspice.

``` bash
mkdir -p auspice/
augur export v2 \
  --tree results/tree.nwk \
  --node-data results/branch_lengths.json \
  --metadata results/subsampled_metadata.tsv \
  --color-by-metadata country region Nextstrain_clade pango_lineage \
  --output auspice/ncov.json
```

We can learn a lot from the tree and its metadata, but we don’t have any details about mutations on the tree, ancestral states, distances between sequences, clades, frequencies of clades through time, etc.
The next set of commands will produce these annotations on the tree in the format of additional node data JSONs.

One of the most important annotations for our analysis is the list of nucleotide and amino acid mutations per branch in the tree.
These annotations allow us to identify putative biologically-relevant mutations and also define clades like those for variants of concern.
To create these annotations, we need to infer the ancestral sequence for each internal node in the tree with `augur ancestral`.
This subcommand is a lightweight wrapper around TreeTime that infers the most likely sequence per position in the given alignment for internal nodes in the given tree.

``` bash
augur ancestral \
  --tree results/tree.nwk \
  --alignment results/aligned.fasta \
  --output-node-data results/nt_muts.json
```

The node data JSON output contains inferred or observed sequences per node and inferred nucleotide mutations per node.
The output also contains the reference’s nucleotide sequence which gets used downstream.

``` bash
less -S results/nt_muts.json
```

We can translate these inferred and observed sequences with `augur translate`, to identify all corresponding amino acid mutations per branch in the tree.

``` bash
augur translate \
  --tree results/tree.nwk \
  --ancestral-sequences results/nt_muts.json \
  --reference-sequence data/genemap.gff \
  --output-node-data results/aa_muts.json
```

The node data JSON output contains gene coordinates in an "annotations" key that will be used by Auspice later on to visualize mutations per gene.

``` bash
less -S results/aa_muts.json
```

With these nucleotide and amino acid mutations per branch of the tree and a predefined list of mutations per clade, we can assign internal nodes and tips to clades.

``` bash
augur clades \
  --tree results/tree.nwk \
  --mutations results/nt_muts.json results/aa_muts.json \
  --clades config/clades.tsv \
  --output-node-data results/clades.json
```

The node data JSON contains a "clade_membership" key for each node in the tree.
Additionally, the first node in the tree for a given clade receives a "clade_annotation" key.
This second key is used to visualize clade names as branch labels in Auspice.

``` bash
less results/clades.json
```

In a similar way that we infer the ancestral nucleotides for each node in the tree at each position of the alignment, we can infer the ancestral states for other discrete traits available in the metadata.
The `augur traits` subcommand is a lightweight wrapper around TreeTime that performs discrete trait analysis (DTA) on columns in the given metadata.
The command assigns the most likely ancestral states to named internal nodes and tips missing values for those states (i.e., samples for which metadata columns contain "?" values) and optionally produces confidence values per possible state.
The following command infers ancestral country and region.

``` bash
augur traits \
  --tree results/tree.nwk \
  --metadata results/subsampled_metadata.tsv \
  --columns country region \
  --confidence \
  --output-node-data results/traits.json
```

Inspect the resulting node data JSON output.
Note that this output also contains the inferred transition matrix and equilibrium probabilities for each requested column.

``` bash
less results/traits.json
```

We now have enough information to investigate mutations in the tree, which geographic locations those mutations might have first appeared in, and how those mutations correspond to known clades in the tree.
We can export these into the Auspice JSON file that Auspice will use to visualize the tree and its annotations.

``` bash
augur export v2 \
  --tree results/tree.nwk \
  --node-data results/branch_lengths.json \
              results/nt_muts.json \
              results/aa_muts.json \
              results/clades.json \
              results/traits.json \
  --metadata results/subsampled_metadata.tsv \
  --color-by-metadata country region Nextstrain_clade pango_lineage \
  --panels tree map entropy frequencies \
  --geo-resolutions country region \
  --include-root-sequence \
  --output auspice/ncov.json
```

In addition to the annotated tree, we often want to know how frequencies of mutations, clades, or other traits change over time.
We can estimate these frequencies with the `augur frequencies` subcommand.
This subcommand assigns a KDE kernel to each tip in the given tree centered on the collection date for the tip in the given metadata.
The command sums and normalizes the KDE values across all tips and at each timepoint ("pivot") such that frequencies equal 1 at all timepoints.
The output JSON file is an Auspice "side-car JSON" that Auspice knows how to load for a given main Auspice JSON based on its filename.
The following command estimates frequencies from the subsampled data at weekly timepoints with a KDE bandwidth of approximately 2 weeks (measured in years).

``` bash
augur frequencies \
  --metadata results/subsampled_metadata.tsv \
  --tree results/tree.nwk \
  --method kde \
  --pivot-interval 1 \
  --pivot-interval-units weeks \
  --narrow-bandwidth 0.041 \
  --proportion-wide 0.0 \
  --output auspice/ncov_tip-frequencies.json
```

To visualize the final tree and frequencies, we can drag these files together onto the [auspice.us](http://auspice.us) landing page or we can run a local Auspice server with the following command.

``` bash
auspice view --datasetDir auspice/
```
