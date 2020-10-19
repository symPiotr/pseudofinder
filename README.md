<p align="center">
<h1 align="center">Pseudofinder</h1>
<h5 align="center">Automated detection of pseudogene candidates in prokaryotic genomes.</h5>
</p>
<br>

## Table of contents
- [Introduction](#introduction)
- [Getting started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [Installation](#installation)
- [Preparing your genome](#preparing-your-genome)
    - [Assembly recommendations](#assembly-recommendations)
    - [Annotation recommendations](#annotation-recommendations)
- [How does Pseudofinder detect pseudogene candidates?](#how-does-pseudofinder-detect-pseudogene-candidates?)
- [Commands](#commands)
    - [Annotate](#annotate)
    - [Reannotate](#reannotate)
    - [Visualize](#visualize)
    - [Test](#test)
- [Versions and changes](#versions-and-changes)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgements](#acknowledgements)
- [References](#references)
- [Wish list](#wish-list)
- [Citing Pseudofinder](#citing-pseudofinder)


## Introduction

Pseudofinder is a Python3 script that detects pseudogene candidates [https://en.wikipedia.org/wiki/Pseudogene] from annotated genbank files of bacterial and archaeal genomes.

It was tested mostly on genbank (.gbf/.gbk) files annotated by Prokka [https://github.com/tseemann/prokka] with the --compliant flag (i.e. including both /gene and /CDS annotations).

There are alternative programs for pseudogene finding and annotation (e.g. the NCBI Prokaryotic Genome Annotation Pipeline [https://www.ncbi.nlm.nih.gov/genome/annotation_prok/]), but to the best of our knowledge, none of them is open source and allows easy fine-tuning of parameters.


## Getting started

These instructions will (hopefully) get you the pipeline up and running on your local machine.

### Prerequisites

Installation requirements: 
- python3
- pip3 (or any other way how to install python libraries, e.g. conda or easy_install)
- biopython
- ncbi-blast+
- Databases: NCBI-NR (non-redundant) protein database (or similar such as SwissProt) formatted for blastP/blastX searches.
- An annotated prokaryotic genome in genbank (.gbf/.gbk) format.

### Easy Installation (Conda required)
This will pseudo-finder into your $PATH and take care of all dependencies
```
git clone https://github.com/filip-husnik/pseudo-finder.git

bash setup.sh

source activate pseudofinder
```


### Installation

A step by step series of commands to install all system dependencies:

<b>Installation of python3, pip3, git (optional), and ncbi-blast+ on Ubuntu (as an administrator):</b>
```
sudo apt-get update

sudo apt-get install python3
sudo apt-get install python3-pip
sudo apt-get install ncbi-blast+
sudo apt-get install git
```

<b>Installation of 3rd party python3 libraries on Ubuntu (as an administrator):</b>
```
sudo pip3 install biopython
sudo pip3 install plotly
sudo pip3 install pandas
sudo pip3 install numpy
sudo pip3 install reportlab
```

*Alternative installation with conda (no root access required) is also possible:*

```
#install Python3 miniconda https://conda.io/miniconda.html
conda install -c conda-forge biopython
conda install -c bioconda blast
conda install plotly
conda install pandas
conda install numpy
conda install reportlab
```

<b>Clone the up to date pseudofinder.py code from github (no root access required):</b>

```
git clone https://github.com/filip-husnik/pseudo-finder.git
```
*Or download a stable release:*

```
# No stable version released yet.
```


## Preparing your genome

### Assembly recommendations

We recommend several rounds of Pilon polishing with Illumina reads to improve your consensus sequence [https://github.com/broadinstitute/pilon/wiki], particularly if you're interested to find pseudogenes in minION/PacBio assemblies. However, Pseudofinder can also help with finding sequencing/basecalling errors potentially breaking genes in MinION/PacBio-only assemblies.

Genomes closed into one or several circular-mapping molecules (chromosomes, plasmids, and phages) should be ideally oriented based on their origin of replication [e.g. by Ori-Finder 2; http://tubic.tju.edu.cn/Ori-Finder2/] to avoid broken genes on contigs randomly linearized by the genome assembler.

### Annotation recommendations

We recommend genbank (.gbf/.gbk) files generated by Prokka [https://github.com/tseemann/prokka] with the --compliant and --rfam flags. Annotating rRNAs, tRNAs, and other ncRNAs in Prokka is recommended to eliminate any false positive 'pseudogene' candidates. ORFs overlapping with non-coding RNAs such as rRNA can be sometimes misannotated in databases as 'hypothetical proteins'.

```
prokka --compliant --rfam contigs.fa
```


## How does Pseudofinder detect pseudogene candidates?

This flowchart shows all steps of the Pseudofinder pipeline.

![alt text](https://github.com/filip-husnik/pseudo-finder/blob/master/flowchart.png)


## Commands


### Annotate

<b>Annotate</b> is Pseudofinder's core command. Calling this command will identify pseudogene candidates in the input genome annotation, and will produce various output files, explained in detail below.

As with any other python script, there are two ways how to run it:
```
# Call it directly with python3 (or just python if python3 is your default).
python3 pseudofinder.py

# Or make the file executable and then rely on its shebang line [#!/usr/bin/env python3].
chmod u+x
./pseudofinder.py
```

Providing input files:
```
# Run the full pipeline on 16 processors (for BlastX/BlastP searches).
# Unless you have a $BLASTDB environmental variable set on your system, you have to provide a full path to the NR database.
python3 pseudofinder.py annotate --genome GENOME.GBF --outprefix PREFIX --database /PATH/TO/NR/nr --threads 16 
```

<b>Output of Annotate:</b>

Every run will produce the following files:

| File | Description |
| --- | --- |
| \[prefix]_intact.gff | Intact genes in GFF3 format. |
| \[prefix]_intact.faa | Intact genes in fasta format. |
| \[prefix]_intergenic.fasta | Intergenic regions in fasta format. |
| \[prefix]_blastX_output.tsv | Tab-delimited output of BLASTX run on intergenic regions. |
| \[prefix]_log.txt | Summary of all inputs, outputs, parameters and results. |
| \[prefix]_map.pdf | Concatenated chromosome map. Input genes appear on the inner track in blue, and candidate pseudogenes are shown in red on the outer track. |
| \[prefix]_proteome.faa | All protein sequences in fasta format. |
| \[prefix]_blastP_output.tsv | Tab-delimited output of BLASTP run on proteome. |
| \[prefix]_pseudos.gff | Candidate pseudogenes in GFF3 format. |
| \[prefix]_pseudos.fasta | Candidate pseudogenes in fasta format. |
| \[prefix]_dnds | Directory containing output from the dnds module: BLAST results, dN/dS summary file, and a folder containing the nucleotide, amino acids, and codon alignments that were used to calculate dN and dS values. |

### dN/dS
The <b>dnds</b> command will compare a genome against another closely-related genome. After homologous genes are identified, this module runs PAML on aligned genes to generate codon alignments and calculate per-gene dN/dS values. These dN/dS values can be used to infer neutral selection and potential cryptic pseudogenes. This module can be invoked withing the <b>Annotate</b> command by providing a closely-related reference genome using the -ref flag.

Usage:
```
pseudofinder.py dnds -a GENOME_PROTS -n GENOME_GENES -ra REFERENCE-PROTS -rn REFERENCE_GENES
``` 


### Reannotate
<b>Reannotate</b> will run the <b>annotate</b> workflow, beginning after the BLAST steps. 
This command can very quickly reannotate pseudogenes if you would like to change any parameters downstream of BLAST.

Usage:
```
pseudofinder.py reannotate -g GENOME -p BLASTP -x BLASTX -log LOGFILE -op OUTPREFIX
``` 

### Visualize

One strength of Pseudofinder is its ability to be fine-tuned to the user's preferences. 
To help visualize the effects of changing the parameters of this program, we have provided the <b>visualize</b> command. 
This command will display how many pseudogenes will be detected based on any combination of '--length_pseudo' and '--shared_hits'. 
It is run by providing the blast files from the <b>annotate</b> command:
```
pseudofinder.py visualize -g GENOME -op OUTPREFIX -p BLASTP -x BLASTX -log LOGFILE
```

### Test

With a single command, the entire Pseudofinder workflow can be run on the 139 kbp genome of <i>Candidatus</i> Tremblaya princeps strain PCIT.

Simply enter the following command:

```
python3 pseudofinder.py test --database /PATH/TO/NR/nr
```
The workflow will begin immediately and write the results to a timestamped folder found in ```/pseudo-finder/test/```.



## Tutorial (Binder)

Initially forked from [here](https://github.com/binder-examples/conda). Thank you to the awesome [binder](https://mybinder.org/) team!

To start the tutorial, hit the 'launch binder' button below, and follow the commands in 'Walkthrough'

[![Binder](https://mybinder.org/badge_logo.svg)](https://gesis.mybinder.org/binder/v2/gh/Arkadiy-Garber/bvcn-binder-pseudofinder/master?urlpath=lab)


### Walkthrough

Print general help menu

    pseudofinder.py help

Print the help menu of the Annotate module

    pseudofinder.py annotate -h

Run the Annotate module

    pseudofinder.py annotate -g test_data/Mycobacterium_leprae_TN.gbff -db test_data/combined_mycobacteria.faa -op testAnnotate

Run the Annotate module, along with the DNDS module

    pseudofinder.py annotate -g test_data/Mycobacterium_leprae_TN.gbff -db test_data/combined_mycobacteria.faa -op testAnnotate --diamond -ref test_data/Mycobacterium_tuberculosis_H37Rv.gbff



## Versions and changes

Read the ChangeLog.txt [https://github.com/filip-husnik/pseudo-finder/blob/master/ChangeLog.txt] for major changes or look at Github commits for everything else [https://github.com/filip-husnik/pseudo-finder/commits/master].


## Contributing

We appreciate any critical comments or suggestions for improvements. Please raise issues or submit pull requests.


## License

This project is licensed under the GNU General Public License v3.0 - see the [LICENSE.md](LICENSE.md) file for details.


## Acknowledgements

This code was inspired mostly by work on bacterial symbionts in early stages of becoming intracellular and strictly host-associated. This ecological shift releases selection pressure ('use it or loose it') on many genes considered essential for free-living bacteria, so relatively recent symbionts can have over 50% of their genes pseudogenized.


## References

**Basic information about bacterial pseudogenes:**

Recognizing the pseudogenes in bacterial genomes: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC1142405/

Taking the pseudo out of pseudogenes: https://www.ncbi.nlm.nih.gov/pubmed/25461580

**Several examples from the Sodalis clade showing how important is pseudogene annotation for bacteria in a nascent stage of symbiosis:**

Mobile genetic element proliferation and gene inactivation impact over the genome structure and metabolic capabilities of Sodalis glossinidius, the secondary endosymbiont of tsetse flies: https://www.ncbi.nlm.nih.gov/pubmed/20649993

A novel human-infection-derived bacterium provides insights into the evolutionary origins of mutualistic insect–bacterial symbioses: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3499248/

Genome degeneration and adaptation in a nascent stage of symbiosis: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC3914690/

Repeated replacement of an intrabacterial symbiont in the tripartite nested mealybug symbiosis: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5027413/

Large scale and significant expression from pseudogenes in Sodalis glossinidius - a facultative bacterial endosymbiont: https://www.biorxiv.org/content/early/2017/07/23/124388

## Wish list
There are several additional features we'll try to include in the script in the near future.

1. Include an optional FPKM cut-off when there are RNA-Seq data available.

2. Improve logic for ORFs on contig ends broken by assembly issues (e.g. metagenome-assembled genomes).

3. Check if the ORFs called as pseudogenes do not represent individual protein domains that can exists and evolve independently of the rest of the original multi-domain protein chain (PFAM?)

4. Fine tune pseudogene finding for mobile elements such as transposases.

5. Visualize results by a scatter plot of all genes/pseudogenes (dN/dS, GC content, expression, length ratio, ...).

6. Sometimes ORFs are predicted by mistake on the opposite strand (e.g. in GC-rich genomes), check regions with ORFS with no blastP hits by blastX.

Please suggest any additional features here: [https://github.com/filip-husnik/pseudofinder/issues].

## Citing Pseudofinder

Pseudofinder is developed by Mitch Syberg-Olsen<sup>1</sup>, Arkadiy Garber<sup>2</sup>, Patrick Keeling<sup>1</sup>, John McCutcheon<sup>2</sup>, and Filip Husnik<sup>3</sup>.

<sup>1</sup> University of British Columbia, Vancouver, Canada

<sup>2</sup> Arizona State University, Tempe, Arizona, USA

<sup>3</sup> Okinawa Institute of Science and Technology, Okinawa, Japan

This project still a work in progress, so there is no official publication. If it was useful for your work, you can cite it as: M. Syberg-Olsen, A. Garber, P. Keeling, J. McCutcheon, & F. Husnik 2020: Pseudofinder, GitHub repository: https://github.com/filip-husnik/pseudofinder/.

Please also cite various dependencies used by Pseudofinder.
