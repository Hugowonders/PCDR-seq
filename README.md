# PCDR-seq
This repository is a supplementary for the manuscript entitled "Characterizing the amplification of STR markers in multiplex polymerase chain displacement reaction using massively parallel sequencing". The scripts below demonstrate how to obtain STR information from pair-end Illumina FASTQ files of PCDR products for each type of amplicons ab initio. First, sequencing quality is checked using [Fastp](https://github.com/OpenGene/fastp). Then, pair-end reads are merged using a modified version of [FLASH 1.2.11](https://github.com/Jerrythafast/FLASH-lowercase-overhang). Next, [SeqKit](https://bioinf.shenwei.me/seqkit) is used to separate different PCDR amplicons from the merged FASTQ file. Finally, STR were genotyped using [FDSTools](https://fdstools.nl/). 

Codes in this script were tested on an AMD RYZEN PC running Ubuntu 20.04 LTS. Other UNIX/Linux systems are plausible but the performance is not guaranteed. Please not that the script is an early-stage implementation, please contact the corresponding author of the manuscript if you encounter any bugs.

## Working directory
Fisrt let's create a directory named "PCDR-seq".
```shell
mkdir ~/PCDR-seq
#set this directory as variable $WD
WD=~/PCDR-seq
```
This is the default working diretory for data storage and analysis. You can also allocate another directory while the filepath should be declared when running the code.

## Software installation
For dependency management, we prefer to use [bioconda](https://anaconda.org/bioconda) for software installation
[Fastp](https://github.com/OpenGene/fastp) is adopted for sequencing quality control.
***
[![install with conda](
https://anaconda.org/bioconda/fastp/badges/version.svg)](https://anaconda.org/bioconda/fastp)
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
cp ./flash $WD
rm -r FLASH-lowercase-overhang
```

[SeqKit](https://bioinf.shenwei.me/seqkit) is used for PCDR amplicon separation through a regular expression matching pipeline. The newest version of Seqkit can be easilyt installed unsing bioconda.
***
[![install with conda](
https://anaconda.org/bioconda/seqkit/badges/version.svg)](https://anaconda.org/bioconda/seqkit)
```shell
conda install -c bioconda seqkit
```

[FDSTools](https://fdstools.nl/) is a software package for forensisc sequencing data analysis. The software is written in Python and can be installed using `pip`. Please make sure Python 3 are available on your system.
```shell
pip install fdstools
```
To install the latest version, you need to make sure you're installing using `pip3` if you have installed both Python 2 and Python 3.

## Sequencing data
FASTQ files of 2800M control DNA amplified by PCDR are accessible in the Sequence Read Archive database (SRR21589082 in BioProject PRJNA858989). We recommed to download the whole archive using [SRA Toolkit](https://github.com/ncbi/sra-tools).
```shell
# download the pre-compiled version
wget https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/3.0.0/sratoolkit.3.0.0-ubuntu64.tar.gz && \
# unzip it
tar zxvf sratoolkit.3.0.0-ubuntu64.tar.gz
rm sratoolkit.3.0.0-ubuntu64.tar.gz
# download fastq files to the working directory using the prefetch utility
./sratoolkit.3.0.0-ubuntu64.tar.gz/bin/prefetch -T C -O $WD && \
# copy fastq files to the working directory
cp $WD/SRR21589082/* $WD
rm -r $WD/SRR21589082
```
In following sections, we would take the two paired PCDR files in the archive, namely 2800M_PCDR1.fastq.gz and 2800M_PCDR2.fastq.gz as an example to introduce the data analysis process in detail.

## Quality control
First we use fastp to remove adaptors and low-quality reads.
```
# activate conda enviroment
conda activate

# quality control of paired fastq files and name them properly.
fastp -i 2800M_PCDR_read1.fastq.gz -I 2800M_PCDR_read2.fastq.gz --dont_eval_duplication \
-o 2800m_rd1.fq.gz -O 2800m_rd2.fq.gz
```

## Merging paired reads
Next we use FLASH to merge paired reads. Note that the `--lowercase-overhang` (`-l`) is used so that truncated STR merging can be identified.
```
# merge paired reads with the miniumum overlap between two reads (`-m`) of 5 bp and the maximum overlap (`-M`) of 250 bp. The maximum mismatch density (`-x`) was set as 0.2. Outies combining (`-O`) is allowed.
./flash 2800m_rd1.fq.gz 2800m_rd2.fq.gz -l -z -c -m 5 -x 0.2 -M 250 -O > 2800m.fastq.gz
```

## PCDR amplicon separation
This process is based on a regular expression matching pipline. Generally, the sequences of four primer sets, namely the forward (FW), reverse (RV), outer forward (OF), and outer reverse (OR) primer are used to match and extract amplicons in turn. Through this pipeline, four different amplicon types can be detected from the PCDR products for each STR, i.e., the shortest amplicon S (FW + RV), the medium amplicons M1 (OF + RV) and M2 (FW + OR), and the longest amplicon L (OF + OR) are seprated in different FASTQ files for futher analysis.

These primer sets are provided in this repository. Directly clone it to the working directroy.
```
git clone https://github.com/Hugowonders/PCDR-seq.git
cp ./PCDR-seq/primer* $WD
rm -r ./PCDR-seq
```

Now we can seprate each amplicon type using [SeqKit](https://bioinf.shenwei.me/seqkit).
```
# make sure all needed files are in the working directroy
cd $WD

fq=2800m.fq.gz
fw=primer1_fw
rv=primer2_rv
of=primer3_of
or=primer4_or

# activate conda environment
conda activate

# write read id into a pool
seqkit seq 2800m.fq.gz -n -o id.pool

# extract amplicon L and write into a new fatq file
seqkit grep -i -s $fq -f $of | seqkit grep -i -s -f $or -o 2800m_L.fastq.gz

# remove amplicon L from original fastq
seqkit seq -n 2800m_L.fastq.gz -o L.id
cat 
```


## Original manuscript
Huang, Yuguo and Zhang, Haijun and Wei, Yifan and Cao, Yueyan and Zhu, Qiang and Li, Xi and Shan, Tiantian and Dai, Xuan and Zhang, Ji, Characterizing the Amplification of STR Markers in Multiplex Polymerase Chain Displacement Reaction Using Massively Parallel Sequencing. Available at SSRN: [https://ssrn.com/abstract=4184676](https://ssrn.com/abstract=4184676)
