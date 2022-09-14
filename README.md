# PCDR-seq
This repository is a supplementary for the manuscript entitled "Characterizing the amplification of STR markers in multiplex polymerase chain displacement reaction using massively parallel sequencing". The scripts below demonstrate how to obtain STR information from pair-end Illumina FASTQ files of PCDR products for each type of amplicons ab initio. First, sequencing quality is checked using [Fastp](https://github.com/OpenGene/fastp). Then, pair-end reads are merged using a modified version of [FLASH 1.2.11](https://github.com/Jerrythafast/FLASH-lowercase-overhang). Next, [SeqKit](https://bioinf.shenwei.me/seqkit) is used to separate different PCDR amplicons from the merged FASTQ file. Finally, STR were genotyped using [FDSTools](https://fdstools.nl/). 

Codes in this script were tested on an AMD Ryzen PC running Ubuntu 20.04. Other UNIX/Linux systems are plausible but the performance is not guaranteed. Please not that the script is an early-stage implementation, please contact the corresponding author of the manuscript if you encounter any bugs.

## Working directory
Fisrt let's create a "PCDR-seq" directory.
```shell
mkdir ~/PCDR-seq
#set this directory as variable $WD
WD=~/PCDR-seq
```
This is the default working diretory for data storage and analysis. You can also allocate another directory while the filepath should be declared when running the code.

## Software installation
For dependency management, we prefer to use [bioconda](https://anaconda.org/bioconda) for software installation
[Fastp](https://github.com/OpenGene/fastp) is adopted for sequencing quality control.
[![install with conda](
https://anaconda.org/bioconda/fastp/badges/version.svg)](https://anaconda.org/bioconda/fastp).
```shell
conda install -c bioconda fastp
```

The modified-version of [FLASH 1.2.11](https://github.com/Jerrythafast/FLASH-lowercase-overhang) is compatible with truncated STR reads merging. It can be directly cloned from the github repository and complied for use.
```shell
git clone https://github.com/Jerrythafast/FLASH-lowercase-overhang.git
# change to the FLASH directory
cd FLASH-lowercase-overhang
# compile the files
make
# now the excutable file named "flash" can be copied to the working directory
cp flash $WD
```

[SeqKit](https://bioinf.shenwei.me/seqkit) is used for PCDR amplicon separation through a regular expression matching pipeline. The newest version of Seqkit can be easilyt installed unsing bioconda
[![install with conda](
https://anaconda.org/bioconda/seqkit/badges/version.svg)](https://anaconda.org/bioconda/seqkit).
```shell
conda install -c bioconda seqkit
```

[FDSTools](https://fdstools.nl/) is a software package for forensisc sequencing data analysis. The software is written in Python and can be installed using `pip`. Please make sure Python 3 are available on your system.
```shell
pip install fdstools
```
To install the latest version, you need to make sure you're installing using `pip3` if you have installed both Python 2 and Python 3.

## Sequencing data
FASTQ files of 2800M control DNA are accessible in the Sequence Read Archive database (SRR20218109 in BioProject PRJNA858989). We recommed to download the whole archive using [SRA Toolkit](https://github.com/ncbi/sra-tools).
```shell
# download the pre-compiled version
wget https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/3.0.0/sratoolkit.3.0.0-ubuntu64.tar.gz && \
# unzip it
tar zxvf sratoolkit.3.0.0-ubuntu64.tar.gz
# download data to the working directory using the prefetch utility
./sratoolkit.3.0.0-ubuntu64.tar.gz/bin/prefetch SRR20218109 -O $WD
```
