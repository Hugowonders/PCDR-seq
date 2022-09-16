# PCDR-seq
This repository is a supplementary for the manuscript entitled "Characterizing the amplification of STR markers in multiplex polymerase chain displacement reaction using massively parallel sequencing". The scripts below demonstrate how to obtain STR information from pair-end Illumina FASTQ files of PCDR products for each type of amplicons ab initio. First, sequencing quality is checked using [Fastp](https://github.com/OpenGene/fastp). Then, pair-end reads are merged using a modified version of [FLASH 1.2.11](https://github.com/Jerrythafast/FLASH-lowercase-overhang). Next, [SeqKit](https://bioinf.shenwei.me/seqkit) is used to separate different PCDR amplicons from the merged FASTQ file. Finally, STRs were genotyped using [FDSTools](https://fdstools.nl/). 

Codes in this script were tested on an AMD RYZEN PC running Ubuntu 20.04 LTS. Other UNIX/Linux systems are plausible but the performance is not guaranteed. Please note that the script is an early-stage implementation. Please contact the corresponding author of the manuscript if you encounter any bugs.

## Working directory
First let's create a directory named "PCDR-seq".
```shell
mkdir ~/PCDR-seq
#set this directory as variable $WD and change into it
WD=~/PCDR-seq
cd $WD
```
This is the default working directory for data storage and analysis in this script. You can also allocate another directory while the path should be declared when running the code.

## Software installation
For dependency management, we prefer to use [bioconda](https://anaconda.org/bioconda) for software installation if available. Other ways to install the software please refer to the webpage of the software.

[Fastp](https://github.com/OpenGene/fastp) is adopted for raw FASTQ files processing.

[![install with conda](
https://anaconda.org/bioconda/fastp/badges/version.svg)](https://anaconda.org/bioconda/fastp)
```shell
conda install -c bioconda fastp
```

The modified version of [FLASH 1.2.11](https://github.com/Jerrythafast/FLASH-lowercase-overhang) is compatible with truncated STR reads merging. It can be directly cloned from the GitHub repository and complied for use.
```shell
git clone https://github.com/Jerrythafast/FLASH-lowercase-overhang.git
# change to the FLASH directory
cd FLASH-lowercase-overhang

# compile the files
make

# now the executable file named "flash" can be copied to the working directory
cp ./flash $WD
cd $WD
rm -rf FLASH-lowercase-overhang
```

[SeqKit](https://bioinf.shenwei.me/seqkit) is used for PCDR amplicon separation through a regular expression matching pipeline. The newest version of Seqkit can be easily installed using bioconda.

[![install with conda](
https://anaconda.org/bioconda/seqkit/badges/version.svg)](https://anaconda.org/bioconda/seqkit)
```shell
conda install -c bioconda seqkit
```

[FDSTools](https://fdstools.nl) is a software package for forensic sequencing data analysis. The software is written in Python and can be installed using `pip`. Please make sure Python 3 is available on your system.
```shell
pip install fdstools
```
To install the latest version, you need to make sure you're installing using `pip3` if you have installed both Python 2 and Python 3.

## Sequencing data
FASTQ files of 2800M control DNA amplified by PCDR are accessible in the Sequence Read Archive database (SRR21589082 in BioProject PRJNA858989). We recommend downloading the whole archive using [SRA Toolkit](https://github.com/ncbi/sra-tools).
```shell
# download the pre-compiled version and unzip it
wget https://ftp-trace.ncbi.nlm.nih.gov/sra/sdk/3.0.0/sratoolkit.3.0.0-ubuntu64.tar.gz && \
tar zxvf sratoolkit.3.0.0-ubuntu64.tar.gz
rm sratoolkit.3.0.0-ubuntu64.tar.gz

# download fastq files to the working directory using the prefetch utility
./sratoolkit.3.0.0-ubuntu64.tar.gz/bin/prefetch -T C -O $WD && \

# copy fastq files to the working directory
cp $WD/SRR21589082/* $WD
rm -r $WD/SRR21589082
```
In the following sections, we would take the two paired PCDR files in the archive, namely 2800M_PCDR1.fastq.gz and 2800M_PCDR2.fastq.gz as an example to introduce the data analysis process in detail.

## Quality control
We use Fastp to remove adaptors and low-quality reads.
```shell
# activate conda enviroment
conda activate

# quality control of paired fastq files and name them properly.
fastp -i 2800M_PCDR_read1.fastq.gz -I 2800M_PCDR_read2.fastq.gz --dont_eval_duplication \
-o 2800m_rd1.fastq.gz -O 2800m_rd2.fastq.gz
```

## Merging paired reads
Next, we use FLASH to merge paired reads. Note that the `--lowercase-overhang` (`-l`) is used so that truncated STR merging can be identified.
```shell
# merge paired reads with the minimum overlap between two reads (`-m`) of 5 bp and the maximum overlap (`-M`) of 250 bp. The maximum mismatch density (`-x`) was set as 0.2. Outies combining (`-O`) is allowed.
./flash 2800m_rd1.fastq.gz 2800m_rd2.fastq.gz -l -z -c -m 5 -x 0.2 -M 250 -O > 2800m.fastq.gz
```

## PCDR amplicon separation
This process is based on a regular expression matching pipeline. Generally, the sequences of four primer sets, namely the forward (FW), reverse (RV), outer forward (OF), and outer reverse (OR) primer are used to match and extract amplicons in turn. Through this pipeline, four different amplicon types can be detected from the PCDR products for each STR, i.e., the shortest amplicon S (FW + RV), the medium amplicons M1 (OF + RV) and M2 (FW + OR), and the longest amplicon L (OF + OR) are separated in different FASTQ files for further analysis.

Primer sets used for amplicon separating and an FDSTools configuration file are provided in this repository. Directly clone it to the working directory.
```shell
git clone https://github.com/Hugowonders/PCDR-seq.git
cp ./PCDR-seq/primer* ./PCDR-seq/fdstools.conf $WD
rm -r ./PCDR-seq
```

Now we can separate each amplicon type using SeqKit.
```
# make sure all needed files are in the working directory
cd $WD

fq=2800m.fastq.gz
fw=primer1_fw
rv=primer2_rv
of=primer3_of
or=primer4_or

# activate conda environment
conda activate

# write read id into a pool
seqkit seq $fq -n -o id.pool

# extract amplicon L using $of and $or and write into a new fastq file
seqkit grep -i -s $fq -f $of | seqkit grep -i -s -f $or -o 2800m_L.fastq.gz

# remove amplicon L using $or from original fastq
seqkit seq -n 2800m_L.fastq.gz -o L.id
cat id.pool L.id | sort -n | uniq -u > id_sub1.pool
seqkit -grep -i -n $fq -f id_sub1.pool -o 2800m_sub1.fastq.gz

# now repeat this procedure to extract other amplicon types

# extract amplicon M2 using $or
seqkit grep -i -s 2800m_sub1.gastq.gz -f $or -o 2800m_M2.fastq.gz

# remove amplicon M2 from sub-fastq
seqkit seq -n 2800m_sub1.fastq.gz -o M2.id
cat id_sub1.pool M2.id | sort -n | uniq -u > id_sub2.pool
seqkit grep -i -n 2800m_sub1.fastq.gz -f id_sub2.pool -o 2800m_sub2.fastq.gz

# extract amplicon M1 using $of
seqkit grep -i -s 2800m_sub2.gastq.gz -f $of -o 2800m_M1.fastq.gz

# remove amplicon M1 from sub-fastq
seqkit seq -n 2800m_sub1.fastq.gz -o M1.id
cat id_sub2.pool M1.id | sort -n | uniq -u > id_sub3.pool
seqkit grep -i -n 2800m_sub2.fastq.gz -f id_sub3.pool -o 2800m_sub3.fastq.gz

# extract amplicon S
seqkit grep -i -s 2800m_sub3.gastq.gz -f $fw | seqkit grep -i -s -f %rv -o 2800m_S.fastq.gz

# remove intermediate files
rm *id* *sub*
```
Now we had four separated FASTQ files storing each type of PCDR amplicon. These files can be further processed for STR analysis.

## STR genotyping
We take the shortest amplicon S as an example. FDSTools is used for STR genotyping in our manuscript.
```shell
conf=fdstools.conf
# designating alleles using the tssv tool
fdstools tssv $conf 2800m_S.fastq.gz | \
# mark stutters using stuttermark, +/- stutter were set as 10% and 30% respectively.
fdstools stuttermark -s=-1:30,+1:10 -l $conf - | \
# convert seuqence alleles to allele name
fdstools seqconvert -l $conf allelename - 2800m.S.str
```
The final output is a plain text file that contains the STR allelic information of 2800M. Other STR files can be analyzed likewise.

## Original manuscript
Huang, Yuguo and Zhang, Haijun and Wei, Yifan and Cao, Yueyan and Zhu, Qiang and Li, Xi and Shan, Tiantian and Dai, Xuan and Zhang, Ji, Characterizing the Amplification of STR Markers in Multiplex Polymerase Chain Displacement Reaction Using Massively Parallel Sequencing. Available at SSRN: [https://ssrn.com/abstract=4184676](https://ssrn.com/abstract=4184676)
