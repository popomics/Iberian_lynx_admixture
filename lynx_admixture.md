---
title: "Iberian lynx admixture analyses"
author: "Axel Barlow"
date: "2023-10-31"
output:
  html_document:
    keep_md: true
---



This is a summary of the workflow for carrying out admixture analysis from Lucena-Perez et al. 2023. The starting point are bam files of *Lynx pardinus* and *Lynx lynx* data mapped to the domestic cat genome. Analyses are restricted to autosomal chromosomes. The following analyses are documented here:

1. Generation of Consensify sequences
2. D statistic tests
3. f-hat tests
4. Phylogenetic test of gene flow direction

## 1. Generation of Consensify sequences

First we carry out a depth analysis on each bam, to determine the integer read depth below the 95th percentile of depth

```bash
# run depth analysis in angsd with filters applied
# you need a file "autosomes.txt" which is just the list of autosomal scaffolds in the reference
angsd -i my_input.bam -doCounts 1 -doDepth 1 -minQ 30 -minMapQ 30 -rf autosomes.txt -out my_output

# angsd doesn't calculate the number of positions with zero reads correctly, 
# so I wrote an R script to sort it out
# you should read the README to see how it works
# clone repo from github
git clone https://github.com/drabarlow/depth_doct.R.git

Rscript depth_doct.R my_output.depthSample

# example output:
Read 41 items
The modal read depth is:  5 
The integer read number below the 95th percentile of depth is:  9 
```

Now we can do allele counts in angsd

```bash
angsd -doCounts 1 -minQ 30 -minMapQ 30 -dumpCounts 3 -rf autosomes.txt -i my_input.bam -out my_counts
```

Now we can compute the Consensify sequences

```bash
# clone repo from github
git clone https://github.com/jlapaijmans/Consensify.git

# run Consensify
# You need a list of the scaffold length as a 2 column tab delimited file, e.g.
# ch1 123456
# ch2 654321
perl Consensify_v0.1.pl my_counts.pos my_counts.counts scaffold_lengths.txt output_Consensify.fa 9
```

## 2. D statistic tests

We make use of the Admixture scripts from James Cahill. You need Consensify fastas from the 3 ingroup and an outgroup

```bash
# clone repo from github
git clone https://github.com/jacahill/Admixture.git

# run the Dstat program
# blocksize is 5 Mb
Dstat P1_Consensify.fa P2_Consensify.fa P3_Consensify.fa outgroup.fa 5000000 > P1_P2_P3_OG.dstat

# parse output to calculate D stat
python D-stat_parser.py P1_P2_P3_OG.dstat

# parse output to calculate standard error
python weighted_block_jackknfie_D.py P1_P2_P3_OG.dstat 5000000
```

The final Dstat results are here `./all_d_lynx.txt`. Plot like so:


```r
# read in data
d <- read.table("./all_d_lynx.txt", header=T)

# set up margins
par(mar=c(4,1,3,10), xpd=TRUE)

#plotting D values
plot(d$Dstat, jitter(d$comp, factor=0.5),
	xlim=c(-0.15,0.3), ylim=c(10,0.5),
	xlab="D value", ylab="", axes=F,
	pch=21, cex=1, lwd=1, bg=ifelse(d$abs.z > 3, "red", "white")
)

# axes
axis(1, at=c(-0.15, 0, 0.15, 0.3), las=1, cex.axis=0.8)
axis(3, at=c(-0.15, 0, 0.15, 0.3), las=1, cex.axis=0.8)
#abline(v=0)
lines(c(0,0), c(0,10.5))

#labels
names=c("D(a_lp_al, a_lp_ca, ll, cat)",
"D(a_lp, a_lp_sm, ll, cat)",
"D(a_lp, c_lp, ll, cat)",
"D(c_lp, c_lp, ll, cat)",
"D(a_ll, c_ll_west, a_lp, cat)",
"D(a_ll, ll_east, a_lp, cat)",
"D(a_ll, c_ll, c_lp, cat)",
"D(c_ll_west, c_ll_east, lp, cat)",
"D(c_ll_west, c_ll_west, lp, cat)",
"D(c_ll_east, c_ll_east, lp, cat)")

ypos=c(1:10)
xpos=rep(c(0.3), each=10)

text(xpos, ypos, labels=names, pos=4, offset=0, cex=0.8, font=1)
```

<img src="lynx_admixture_files/figure-html/unnamed-chunk-1-1.png" width="80%" style="display: block; margin: auto;" />

## 3. f-hat tests

We again make use of the Admixture scripts from James Cahill. You need Consensify fastas from 4 ingroup and an outgroup

```bash
Fhat P1_Consensify.fa P2_Consensify.fa P3_Consensify.fa P4_Consensify.fa outgroup.fa 5000000 > P1_P2_P3_P4_OG.fhat

# parse output to calculate D stat
python fhat_parser.py P1_P2_P3_P4_OG.fhat

# parse output to calculate standard error
python weighted_block_jackknfie_fhat.py P1_P2_P3_P4_OG.fhat 5000000
```

The final f-hat results are here `./*_parsed. Plot like so:


```r
# read in data
al <- read.table("./a_lp_al_1311_parsed", header=T)
ca <- read.table("./a_lp_ca_1309_parsed", header=T)
sm <- read.table("./a_lp_sm_1310_parsed", header=T)

# margins
par(mar=c(6,5,1,1))

# plotting fhats
plot(jitter(al$comp, factor=0.5), al$fhat,
	xlim=c(1,30), ylim=c(0,0.03),
	xlab="", ylab="Admix fraction",
	axes=FALSE, frame.plot=TRUE,
	pch=21, cex=0.8, lwd=0.1, bg="red"
)

points(jitter(ca$comp, factor=0.5), ca$fhat,
	pch=21, cex=0.8, lwd=0.1, bg="blue"
)

points(jitter(sm$comp, factor=0.5), sm$fhat,
	pch=21, cex=0.8, lwd=0.1, bg="yellow"
)

# define sample names
names =c(
"c_lp_do_0007",
"c_lp_do_0141",
"c_lp_do_0144",
"c_lp_do_0153",
"c_lp_do_0162",
"c_lp_do_0163",
"c_lp_do_0173",
"c_lp_do_0300",
"c_lp_do_0333",
"c_lp_do_0335",
"c_lp_do_0443",
"c_lp_do_0444",
"c_lp_sm_0134",
"c_lp_sm_0138",
"c_lp_sm_0140",
"c_lp_sm_0155",
"c_lp_sm_0156",
"c_lp_sm_0161",
"c_lp_sm_0185",
"c_lp_sm_0186",
"c_lp_sm_0206",
"c_lp_sm_0208",
"c_lp_sm_0213",
"c_lp_sm_0226",
"c_lp_sm_0276",
"c_lp_sm_0298",
"c_lp_sm_0320",
"c_lp_sm_0325",
"c_lp_sm_0359",
"c_lp_sm_0450"
)

# axes
axis(1, at=c(1:30), las=3, cex.axis=0.8, labels=names)
axis(2, at=c(0, 0.01, 0.02, 0.03), las=1, cex.axis=0.8)
```

<img src="lynx_admixture_files/figure-html/unnamed-chunk-2-1.png" width="100%" style="display: block; margin: auto;" />

## 4. Phylogenetic test of gene flow direction

We make use of the treehaker script. You should read the accompanying documention and ensure required software is installed on your system

```bash
# clone repo from github
git clone https://github.com/jlapaijmans/treehacker.git

# first make fasta for each individual using single read sampling in angsd
angsd -doFasta 1 -minQ 30 -minMapQ 30 -rf autosomes.txt -i my_input.bam

# tree hacker needs a list of fasta names. The outgroup should be the last individual, e.g.
ls fasta.fa | cut -d'.' -f1 > fastanames.txt

# then run treehacker with the following command line variables
# $1 fastanames.txt
# $2 output base name 
# $3 window size
bash treehacker fastanames.txt my_output 100000

# example output: you get the counts of the observed topologies
  21470 (Felis_catus_autosomes,((a_lp_ca_1309,c_lp_do_0007),(c_ll_ba_0227,c_ll_ya_0138)));
    313 (Felis_catus_autosomes,(((a_lp_ca_1309,c_lp_do_0007),c_ll_ba_0227),c_ll_ya_0138));
    167 (Felis_catus_autosomes,(a_lp_ca_1309,((c_ll_ba_0227,c_ll_ya_0138),c_lp_do_0007)));
    161 (Felis_catus_autosomes,(((a_lp_ca_1309,c_lp_do_0007),c_ll_ya_0138),c_ll_ba_0227));
     78 (Felis_catus_autosomes,((a_lp_ca_1309,(c_ll_ba_0227,c_ll_ya_0138)),c_lp_do_0007));
     19 (Felis_catus_autosomes,(((a_lp_ca_1309,c_ll_ba_0227),c_lp_do_0007),c_ll_ya_0138));
     15 (Felis_catus_autosomes,(a_lp_ca_1309,((c_ll_ba_0227,c_lp_do_0007),c_ll_ya_0138)));
     13 (Felis_catus_autosomes,((a_lp_ca_1309,(c_ll_ba_0227,c_lp_do_0007)),c_ll_ya_0138));
     11 (Felis_catus_autosomes,(a_lp_ca_1309,(c_ll_ba_0227,(c_ll_ya_0138,c_lp_do_0007))));
     10 (Felis_catus_autosomes,((a_lp_ca_1309,c_ll_ba_0227),(c_ll_ya_0138,c_lp_do_0007)));
      5 (Felis_catus_autosomes,(((a_lp_ca_1309,c_ll_ba_0227),c_ll_ya_0138),c_lp_do_0007));
      4 (Felis_catus_autosomes,((a_lp_ca_1309,(c_ll_ya_0138,c_lp_do_0007)),c_ll_ba_0227));
      3 (Felis_catus_autosomes,((a_lp_ca_1309,c_ll_ya_0138),(c_ll_ba_0227,c_lp_do_0007)));
      1 (Felis_catus_autosomes,(((a_lp_ca_1309,c_ll_ya_0138),c_lp_do_0007),c_ll_ba_0227));
```

The final results are here ./lynx30. Plot like so:


```r
# I make use of the dplyr library to sum the topology classes informative on admixture and calculate ratios
#library(dplyr)

# get data
mytr <- read.table("./lynx30", header = TRUE)

# calculate ratios
mytr <- mytr %>% mutate(ll_into_lp = ((class4 + class9 + class12) / (class5 + class7 + class11)))
mytr <- mytr %>% mutate(lp_into_ll = ((class2 + class6 + class8) / (class3 + class10 + class13)))

# margins
par(mar=c(3,1,2,1))

# plot
plot(mytr$ll_into_lp, jitter(rep(1, 30)),
	axes=FALSE, xlab="", ylab="",
	ylim=c(0.9,1.15), xlim=c(-1,3),
	pch=21, bg="salmon"
)
arrows(0,1.12,2,1.12, code="3", cex=0.8)
points(1,1.12, pch=19, cex=1.2)
axis(1, at=c(-1:3), labels=c("3x", "2x", "equal", "2x", "3x")) 
```

<img src="lynx_admixture_files/figure-html/unnamed-chunk-3-1.png" width="100%" style="display: block; margin: auto;" />

