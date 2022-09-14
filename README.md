# PCDR-seq
This repository is a supplementary for the manuscript entitled "Characterizing the amplification of STR markers in multiplex polymerase chain displacement reaction using massively parallel sequencing". The scripts below demonstrate how to obtain STR information from pair-end Illumina FASTQ files of PCDR products for each type of amplicons ab initio. First, sequencing quality is checked using [Fastp](https://github.com/OpenGene/fastp). Then, pair-end reads are merged using a modified version of [FLASH 1.2.11](https://github.com/Jerrythafast/FLASH-lowercase-overhang). Next, [Seqkit](https://bioinf.shenwei.me/seqkit) is used to separate different PCDR amplicons from the merged FASTQ file. Finally, STR were genotyped using [FDSTools](https://fdstools.nl/). 

Codes in this script were tested on a AMD Ryzen PC running Ubuntu 20.04. Other UNIX/LINUX platforms are plausible but the performance is not guaranteed. Please not that the script is an early-stage implementation, please contact the communication author of the manuscript if you encounter any bugs.

## Software installation
First, [Fastp](https://github.com/OpenGene/fastp) is adopted for sequencing quality control. For dependencies management, we recommend to install it using Bioconda.
[![install with conda](
https://anaconda.org/bioconda/fastp/badges/version.svg)](https://anaconda.org/bioconda/fastp)
