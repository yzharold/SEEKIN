# SEEKIN software

### SEEKIN:SEequence-based Estimation of KINship.

#### Author: Jinzhuang Dou  <douj@gis.a-star.edu.sg>, Chaolong Wang <wangcl@gis.a-star.edu.sg>

#### License: GNU General Public License v3.0 (GPLv3)
---

## 1 Description
SEEKIN is a software program for estimating pairwise kinship coefficients using either shallow sequencing data or genotyping data. The method was initially developed for shallow sequencing data, such as off-target sequencing data from target sequencing experiments (typically ~0.1-1X). SEEKIN, together with the LASER (use URL here) software [1] for ancestry estimation, enables control of population structure and cryptic relatedness in target/exome sequencing studies.
 
To address the missing data and genotype uncertainty issues that are intrinsic to shallow sequencing data, we use the LD-based genotype calling algorithm in BEAGLE (use URL here) [2] to process sequencing data and develop kinship estimators that explicitly model the genotype uncertainty via the Rsq metric output by BEAGLE.

SEEKIN includes a homogeneous estimator for samples from the sample population (SEEKIN-hom) and a heterogeneous estimator for samples with population structure and admixture (SEEKIN-het). For SEEKIN-het, we use the LASER program to estimate individual ancestry for each study individual, and then derive individual-specific allele frequencies for kinship estimation in a way similar to existing methods [3,4].

Our SEEKIN estimators are also applicable to high-quality genotyping data, such as those from array-genotyping or deep whole-genome sequencing studies. In this special case of no genotype uncertainty, SEEKIN performs similarly to existing kinship estimation programs. 

In terms of implementation, SEEKIN utilizes a "single producer/consumer" design for parallel computation and takes the standard gzipped VCF file as the input.  SEEKIN is both computationally and memory efficient and is scalable to estimate pairwise kinship between 10,000s of individuals on a typical computing cluster.

If you have any bug reports or questions, please send an email to Jinzhuang Dou at douj@gis.a-star.edu.sg or Chaolong Wang at wangcl@gis.a-star.edu.sg.

## 2 Citation for SEEKIN 

Details of our SEEKIN method can be found at:
Dou J et al. Estimation of kinship coefficients using sparse sequencing data. (Manuscript submitted)


## 3 Dependencies
* gcc >= 4.9
* OpenBLAS 
* Armadillo

## 4 Download and install

You can download SEEKIN by typing the following command in a terminal.

`git clone https://github.com/jinzhuangdou/SEEKIN.git` 

This command will create a folder named "SEEKIN" in the current directory. The downloaded package contains a statically linked binary executable seekin (in the bin/ folder), which was pre-compiled and tested in 64-bit Linux machine.  

If you want to compile your own version, please enter the src/ folder, change the library paths in the beginning of the Makefile accordingly, and type `make` 
to compile.

`cd SEEKIN/src && make`

## 5 Usage 
You can type the following command to get the list of help option.
`seekin -h`  

SEEKIN provides three modules 

* **modelAF** for calculating the PC-related regression coefficients of reference samples;
* **getAF** for estimating the individual allele frequencies of study samples; 
* **kinship** for estimating kinship coefficients for samples from either homogenous or heterogenous samples.  

To get the detailed meaning of option for one module (for example `kinship`), you can type: `seekin kinship -h`  

## 6 Examples

Here we provide demo of SEEKIN based on data provided in the `example` folder, which include:

* **Study.10K.vcf.gz**   This file includes genotypes of 10,000 randomly selected SNP for 50 Chinese and 50 Malays. The genotypes were called from off-target data in a WES study using BEAGLE with a reference panel from the 1000 Genome Project. This example dataset is highly noisy and is only for illustration purpose. 
* **SGVP.12K.vcf.gz**   This file includes array genotyping data on 12,000 SNPs, including xx SNPs overlapping with the Study.10K.vcf.gz file. This dataset is a subset of the data from the Singapore Genome Variation Project (SGVP). This dataset is intended to be the ancestry reference panel for estimating individual-specific allele frequencies for SEEKIN-het. 
* **SGVP.RefPC.coord**   This file contains PCA coordinates of the top 10 PCs for the reference individuals in SGVP.12K.vcf.gz. This file was generated by the LASER (----) software. 
* **Study.onSGVP.PC.coord**   This file contains the top 2 PCs for the study individuals in the SGVP reference ancestry space (SGVP.RefPC.coord). This file can be prepared using the LASER software with either sparse sequence reads or array genotyping data. 

  
#### 6.1 SEEKIN-hom: kinship estimation for homogenous samples

When assuming no population structure, we only need the genotype file of study samples (Study.10K.vcf.gz) to estimate kinship using SEEKIN.   

  ```./seekin kinship -i ./Study.10K.vcf.gz  -r 0.3  -m 0.05   -d DS  -p homo  -l 2000  -t 3  -w  1 -o Study.homo``` 
The meaning of all command line options are listed below:
[Given a screen shot of the options above] 

The output includes 5 files with prefixes `Study.chr22.homo` specified by `-o` flag. 

*  _.log

This is the log file to monitor and record the progress for each module when running SEEKIN.

*  _.kin 

This file provides the kinship estimation for all pairs of individuals.The columns are individual IDs for the first and second individual of pair, number of SNPs used, and the estimated kinship coefficient, respectively. Example: 

  ```
  Ind1    Ind2    NSNP    Kinship      
  S1      S2      8592    0.0231     
  S1      S3      8592    0.0370        
  S1      S4      8592    0.0168      
  ```
  
*  _.inbreed 

This file provides the estimated inbreeding coefficient for each individual. The first column is individual ID and the second column is the estimated inbreeding coefficient. Example:
  ```
  Ind1    Inbreed_coef
  S1       0.0296
  S2       0.0228
  S3      -0.0338
  ```
  
*  _.matrix and _.matrixID 

The `_.matrix` file contains an N × N matrix of 2pi, where phi is the estimated kinship matrix of N study individuals. The `.matrixID` file contains the individual IDs corresponding to the order in the `_.matrix` file (also the same order as in the VCF file).


#### 6.2 SEEKIN-het: kinship estimation for heterogenous samples with population structure and admixture

To estimate kinship coefficients for samples with population structure and admixture, we need to first estimate the ancestry background of each individual and derive individual-specific allele frequencies. We use LASER to estimate individual ancestry background, which applicable to both shallow sequencing data and genotyping data under a unified framework given an external ancestry reference panel [1].  Details of LASER can be found on the LASER website (URL). Here, we assume the ancestry coordinates for both the reference individuals and the study individuals have been properly generated by LASER.

* 6.2.1 Model allele frequencies 

We model allele frequencies as linear functions of PCs based on the ancestry reference panel. The regression coefficients can be estimated using the ```modelAF``` module in SEEKIN:

  ```seekin modelAF -i SGVP_268.chr22.vcf.gz -c SGVP_268.chr22.RefPC.coord -k 2 -o SGVP_268.chr22.beta```
  
This command will generate a file, named XXXX (specified by –o), which contains the intercepts and regression coefficients of first two PCs (specified by -k) based on genotypes (specified by -i) and PC coordinates (specified by -c) of the reference individuals. In this file, the first five columns are chromosome, position, reference allele, alternative allele, and allele frequencies of the alternative allele in the reference dataset, respectively. The remaining columns are the estimated intercept and regression coefficients for each PC. This file is tab-delimited. Example:  

  ```
  CHROM   POS     REF     ALT      AF       beta0   beta1   
  10     60969    C       A        0.48     0.96    -0.00    
  10     70969    G       A        0.41     0.96    -0.10    
  ```
* 6.2.2 Estimate individual-specific allele frequencies  

The individual specific allele frequencies of study individuals can be generated by the following command:

  ```
  seekin getAF -i Study.onSGVP.PC.coord -b  SGVP_268.chr22.beta  -k 2  -o Study.chr22.indvAF.vcf
  ```
This command estimates individual-specific allele frequencies based on the top 2 PCs (specified by –k) in the coordinate file (specified by -i) and the regression coefficients (specified by –b). The outputs are stored in a gzipped VCF file (specified by –o). Example:

  ```
  ##fileformat=VCFv4.2
  ##INFO=<ID=AF,Number=A,Type=Float,Description="Estimated allele frequencies averaged across all individuals">
  ##FORMAT=<ID=AF1,Number=A,Type=Float,Description="Estimated individual-specific allele frequencies">
  #CHROM  POS     ID      REF     ALT     QUAL    FILTER  INFO    FORMAT  ID1   ID2  ID3
  1       11008   .       C       G       .       .       AF=0.0500       AF1     0.0534  0.0455  0.0536
  1       12001   .       A       G       .       .       AF=0.0200       AF1     0.0231  0.4451  0.1537
  1       13102   .       C       T       .       .       AF=0.4500       AF1     0.2514  0.0123  0.0216
  1       14052   .       C       G       .       .       AF=0.6500       AF1     0.0524  0.0252  0.9531
  ```

6.2.3 Estimate kinship coefficients for heterogeneous samples 

With the estimated individual-specific allele frequencies, `Stdudy.chr22.indvAF.vcf.gz`, we can run the kinship module to estimate kinship coefficients using the following command:

  ```
  seekin kinship -i ./Study.chr22.vcf.gz  -a  ./Study.chr22.indvAF.vcf.gz  -r 0.3  -m 0.05   -d DS  -p admix -n 2000  -t 3 -w 1  -o Study.chr22.admix
  ```
  
Note that it is important to set –p to “het” so that the SEEKIN-het estimator will be used and the –f option will be effective to take the individual-specific allele frequencies as input.
The output files have the same format as those described in section 6.1.



## 6 Reference

1.  Browning, B.L. & Browning, S.R. A unified approach to genotype imputation and haplotype-phase inference for large data sets of trios and unrelated individuals. Am J Hum Genet 84, 210-23 (2009).
2.  Wang, C. et al. Ancestry estimation and control of population stratification for sequence-based association studies. Nat Genet 46, 409-15 (2014)
3. 
