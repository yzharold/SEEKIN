# SEEKIN software

### Sequence-based Estimation of Kinship and Inbreeding.

#### author: Jinzhuang Dou  <douj@gis.a-star.edu.sg>, Chaolong Wang <wangcl@gis.a-star.edu.sg>

#### license: GNU General Public License v3.0 (GPLv3)
---

## 1 Description

SEEKIN is a software program for estimating kinship and inbreeding coefficients for samples which are sequenced at low sequencing coverage (typically lower than 1x). The key features of this program include:   								
* Account for the genotype uncertainties by leveraging the haplotype information from study or external samples.  
* Estimate kinship for heterogenous samples with population structure and admixture.    
* Analyze thousands of individuals in saving memory usage and computational time by utilizing the "single producer/consumer" design.

## 2 Dependencies
* gcc >= 4.9
* OpenBLAS
* Armadillo

## 3 Download and install

`git clone https://github.com/jinzhuangdou/SEEKIN.git` 

The download package contains a standalone (i.e., statically linked) 64-bit Linux executable seekin (in the `bin/`), which has already been tested on Linux server. If you want to compile your own version of SEEKIN, enter the `src/` folder and type `make` to compile the programs. Before that, you will need to change the library paths in the Makefile accordingly.

`cd SEEKIN/src && make`

## 4 Quick start: 

`cd example && bash runSEEKIN.sh`

## 5 Usage 
You can type the following command to get a list of help option.
`seekin –h`  

SEEKIN provides three modules 

* **modelAF** for calculating the PC-related regression coefficients of reference samples;
* **getAF** for estimating the individual allele frequencies of study samples; 
* **kinship** for estimating kinship coefficients for samples from either homogenous or heterogenous samples.  

To get the detailed list of option for one module (for example `kinship`), you can type: `seekin kinship –h`  

## 6 Examples

* 6.1 Kinship estimation for homogenous samples

Here we provide example usages based on the data provided in the folder named `example`. In this folder, we have the `Study.chr22.vcf.gz` file which includes genotypes of chromosome 22 for 10 studied samples. You may use the following command:
The command line for running SEEKIN for homogenous samples is very simple.   

  ```./seekin kinship -i ./Study.chr22.vcf.gz  -r 0.3  -m 0.05   -d DS  -p homo  -n 2000  -t 3  -w  1 -o Study.chr22.homo``` 
  
It will generate result files with prefixes `Study.chr22.homo` specified by `–o` flag in the current directory. The detailed meanings of flags of `kinship` module are summarized below.  

  ```
  -i Specify the name of the genotype input file of study samples. SEEKIN only reads compressed (gzipped) VCF files. [no default value]
  -a Specify the name of the individual allele frequency file of study samples. SEEKIN only reads compressed (gzipped) VCF files. Note that this option cannot be used for homogenous estimation. [no default value]
  -r Remove sites with Rsq less than the '-r' value. [default  0.3]  
  -m Remove sites with MAF less than the '-m' value. [default 0.05]  
  -d Specify the kinship estimation based on observed or imputed genotypes. It is 'GT' if using observed genotypes and 'DS' using imputed genotypes. If no DS information is available for a marker in the VCF file, the GT filed will be used even though the option –d is set to DS. If both GT and DS information available, we recommend using DS mode, because our model could account for genotype uncertainty effectively. [default DS]  
  -p Specify the population mode when estimating kinship. It is 'homo' for homogenous estimation and 'admix' for admixture estimation.  Note that the option –a must be available when option –p is set to admix. [default  homo]  
  -n Specify the number of markers to include in each block for kinship calculation at one time. This option must be no more than the total number of markers in the input VCF file. [default 10,000]  
  -t Specify the number of threads of execution. [default 1]  
  -w Specify the weight scheme when combing genome-wide markers. 1: 2pqRsq^2; 2: 2Rsq^2 [default 1]  
  -o Specify the output file name prefix. The prefix may be an absolute or relative filename, but it cannot be a directory name.  
  ```
  
* 6.2 Kinship estimation of samples with population structure and admixture 

Except for the genotype files, we also have two PCA coordinate files in a plain text format: 1) `SGVP_268.chr22.RefPC.coord` file which contains PCA coordinates for the top 2 PCs of the reference individuals; 2) `Study.chr22.ProPC.coord` file which contains the top 2 PCs calculated by projecting the study samples on the reference panel using LASER.  In this example, we use the SGVP as reference panel to model the PC-related linear regression coefficients based on the ```modelAF``` module:

  ```seekin modelAF –i SGVP_268.chr22.vcf.gz –c SGVP_268.chr22.RefPC.coord -k 2 –o SGVP_268.chr22.beta```
  
Detailed meanings of flags of `modelAF` module are summarized below.   
  
  ```
  -i Specify the name of the genotype input file of reference samples. SEEKIN only reads compressed (gzipped) VCF files. [no default value] 
  -c Specify the name of PCA coordinate file of reference samples. [no default value]
  -k Specify the number of PCs to compute. This number should be no more than the number of PCs in the input PCA coordinate file. [default 2]. 
  -o Specify the output file name. [no default value]
  ```
  
Using the `SGVP268.beta` file generated above, you may use the following command to estimate the individual allele frequencies of study samples: 

  ```
  seekin getAF –i Study.chr22.ProPC.coord -b  SGVP_268.chr22.beta  -k 2 –o Study.chr22.indvAF.vcf
  ```
  
In the above command, the `Study.chr22.ProPC.coord` file is the projected PCA coordinate file of the study samples on the reference panel, and we recommend the users to read the LASER documentation[2] carefully for more information. The detailed meanings of flags of `getAF` module are summarized below. 

  ```
  -i Specify the PCA coordinate file of study samples. [no default value] 
  -b Specify the PC-related regression coefficient file of reference samples. [no default value]
  -k Specify the number of PCs to compute. This number should be no more than the number of PCs in the input PCA coordinate file. [default 2]. 
  -o Specify the output file name. No need to add the ".gz" suffix. [no default value]
  ```
  
Finally, we can run the `kinship` module with the above `Stdudy.chr22.indvAF.vcf.gz` as the input, the command is: 

  ```
  seekin kinship -i ./Study.chr22.vcf.gz  -a  ./Study.chr22.indvAF.vcf.gz  -r 0.3  -m 0.05   -d DS  -p admix -n 2000  -t 3 -w 1  -o Study.chr22.admix
  ```
  
The command is similar with the homogenous case but requiring the estimated individual allele frequency file specified by the flag `–a` and using the `admix` mode specified by flag `–p`. 


## 7 Output
All output file will be saved in the current directory unless the path to a different directory given in the parameter value. We first describe the 5 output files from the kinship module which will be start with the prefix specified by the `-o` value.

* 7.1 _.log and terminal outputs 

The terminal outputs are used to monitor and record the progress for each module when running SEEKIN. The log file is identical to the terminal outputs. 

* 7.2 _.kin 

This file provides the kinship estimation for all pairs of individuals.The columns are individual ID for the first and second individual of pair, number of SNPs used, the estimated kinship coefficient, respectively. One example is as following: 
  ```
  Ind1    Ind2    NSNP    Kinship      
  S1      S2      8592    0.0231     
  S1      S3      8592    0.0370        
  S1      S4      8592    0.0168      
  ```
  
* 7.3 _.inbreed 

This file provides the estimation of inbreeding coefficient estimation for each individual. The columns are individual ID and inbreeding coefficient, respectively. One example is as following:
  ```
  Ind1    Inbreed_coef
  S1       0.0296
  S2       0.0228
  S3      -0.0338
  ```
  
* 7.4 _.index and _.matrix 

The `_.matrix` file contains an N × N matrix (The variable N here means the sample size of study samples) of estimated kinship coefficients with the corresponding index of each individual shown in `_.index` file.  For example, the kinship coefficient value given in row 2 and column 3 in the `_.matrix` file would correspond to the individuals in the `_.index` file who have indices of 2 and 3, respectively.

* 7.5 PC-related regression coefficient file of reference samples    

Use the `modelAF` module in SEEKIN, you will generate the file which contains the PC-related regression coefficients for reference samples. From the first column to the fifth column are chromosome ID, genome position, reference allele, alternative reference allele, and allele frequencies of non-ref allele, respectively. And the remaining columns are the estimated coefficients for each PC. This file is tab-delimited. An example is as following: 
  ```
  CHROM   POS     REF     ALT      AF       beta0   beta1   
  10     60969    C       A        0.48     0.96    -0.00    
  10     70969    G       A        0.41     0.96    -0.10    
  ```
  
* 7.6 Individual allele frequency file of study samples  

Use the `getAF` module in SEEKIN, you will generate the file which contains the individual allele frequencies for study samples. The generated file is the standard VCF in the compressed format as following: 
  ```
  ##fileformat=VCFv4.2
  ##INFO=<ID=AF,Number=A,Type=Float,Description="Estimated Allele Frequencies of all samples">
  ##FORMAT=<ID=AF1,Number=A,Type=Float,Description="Estimated individual specific Allele Frequencies">
  #CHROM  POS     ID      REF     ALT     QUAL    FILTER  INFO    FORMAT  ID1   ID2  ID3
  1       11008   .       C       G       .       .       AF=0.0500       AF1     0.0534  0.0455  0.0536
  1       12001   .       A       G       .       .       AF=0.0200       AF1     0.0231  0.4451  0.1537
  1       13102   .       C       T       .       .       AF=0.4500       AF1     0.2514  0.0123  0.0216
  1       14052   .       C       G       .       .       AF=0.6500       AF1     0.0524  0.0252  0.9531
  ```
## 8 Reference

```
1. Browning, B.L. & Browning, S.R. A unified approach to genotype imputation and haplotype-phase inference for large data sets of trios and unrelated individuals. Am J Hum Genet 84, 210-23 (2009).
2. Wang, C. et al. Ancestry estimation and control of population stratification for sequence-based association studies. Nat Genet 46, 409-15 (2014).
3. Teo, Y.Y. et al. Singapore Genome Variation Project: a haplotype map of three Southeast Asian populations. Genome Res 19, 2154-62 (2009).
4. Sudmant, P.H. et al. An integrated map of structural variation in 2,504 human genomes. Nature 526, 75-81 (2015).
5. International HapMap, C. A haplotype map of the human genome. Nature 437, 1299-320 (2005).
6. Conomos, M.P., Reiner, A.P., Weir, B.S. & Thornton, T.A. Model-free Estimation of Recent Genetic Relatedness. Am J Hum Genet 98, 127-48 (2016).
```
