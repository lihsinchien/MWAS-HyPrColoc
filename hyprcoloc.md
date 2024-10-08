HyPrColoc
================
Christopher N Foley & James R Staley
2024-09-01

- [1 Introduction](#1-introduction)
  - [1.1 Installation](#11-installation)
- [2 Getting started](#2-getting-started)
  - [2.1 Basic set-up: assuming independence between
    studies](#21-basic-set-up-assuming-independence-between-studies)
  - [2.2 Computing a credible set of snps for each cluster of
    colocalized
    traits](#22-computing-a-credible-set-of-snps-for-each-cluster-of-colocalized-traits)
- [3 An analysis protocol: assessing stability of clusters via a
  sensitivity
  analysis](#3-an-analysis-protocol-assessing-stability-of-clusters-via-a-sensitivity-analysis)
  - [3.1 Assessing sensitivity to changes in the prior configuration
    parameters](#31-assessing-sensitivity-to-changes-in-the-prior-configuration-parameters)
  - [3.2 The Bayesian divisive clustering
    algorithm](#32-the-bayesian-divisive-clustering-algorithm)
- [4 Mapping pleiotropy: an alternative to
  PheWAS](#4-mapping-pleiotropy-an-alternative-to-phewas)
- [5 Analysing large numbers of
  traits](#5-analysing-large-numbers-of-traits)
- [6 Analysing correlated traits](#6-analysing-correlated-traits)

# 1 Introduction

HyPrColoc is Bayesian divisive clustering algorithm for identifying
clusters of traits which colocalize at distinct causal variants in a
genomic region. The default algorithm can identify clusters of
putatively colocalized traits within a vast collection of traits,
e.g. 1000s, quickly. For each set of putatively colocalized traits, the
algorithm outputs: the names of the traits, the posterior probability of
colocalization, the location of the shared putatively causal variant and
a ‘fine-mapping’ probability to quantify evidence supporting the
candidate causal variant being the causal variant.

Traits can be either continuous, e.g. blood pressure, or discrete,
e.g. a disease. Basic analyses require information on the summarized
effect estimates (i.e. estimated regression coefficients) and their
corresponding standard errors for each snp in the genomic region and
each trait under consideration. These should be entered as numeric
matrices. If the traits are measured in non-independent studies
(i.e. those containing overlapping participants) analyses can be
adjusted to account for this. To do this three additional matrices are
required: (i) the pair-wise marginal correlations between the traits;
(ii) the pair-wise LD estimates between the snps in the region and;
(iii) the pair-wise estimates of the proportion of sample overlap
between study participants. We recommend reading our note (below and in
more detail our paper) on adjusting analyses to account for correlated
summary data and a-priori trait correlation before doing so in your
analyses.

## 1.1 Installation

``` r
install.packages("devtools", repos='http://cran.us.r-project.org')
library(devtools)
install_github("cnfoley/hyprcoloc")
```

# 2 Getting started

In the first part of this exercise we begin by loading the package and
some data needed to run analyses using HyPrColoc. For a given region, a
standard analysis requires data from two matrices, of equal size,
denoting: (i) a matrix of effect estimates (betas), with the columns
denoting the study traits and rows the snps, and; (ii) a matrix of
corresponding standard errors (ses).

``` r
library(hyprcoloc)
betas <- hyprcoloc::test.betas
head(betas)
#>                      T1           T2            T3           T4           T5
#> rs6694014    0.02791630 -0.030353270 -0.0006508550 -0.015820079  0.029344113
#> rs11206477  -0.01333565 -0.007925434 -0.0220791782 -0.024533654 -0.006170044
#> rs978479     0.02789307 -0.031569213  0.0013910408 -0.016449156  0.030562284
#> rs6684892    0.01109516 -0.035811867 -0.0041582437 -0.001093336  0.021029899
#> rs149881092 -0.02058398  0.033028644  0.0732322501 -0.062564186  0.031519527
#> rs2081705    0.02804342 -0.031658194 -0.0006283532 -0.017953736  0.029719314
#>                      T6          T7          T8            T9         T10
#> rs6694014   -0.03739309 -0.05015619 -0.03963766 -0.0497666615 -0.01565694
#> rs11206477   0.08915364  0.08788543  0.07824464 -0.0364999498 -0.07140001
#> rs978479    -0.03667044 -0.04993080 -0.03999140 -0.0485026083 -0.01742117
#> rs6684892   -0.04342024 -0.05462565 -0.02610322 -0.0516383902 -0.01373675
#> rs149881092  0.03339484  0.08794777  0.06747322  0.0002877561 -0.01701753
#> rs2081705   -0.03798465 -0.05052849 -0.04150411 -0.0498085699 -0.01767236
ses <- hyprcoloc::test.ses
head(ses)
#>                     T1         T2         T3         T4         T5         T6
#> rs6694014   0.01640805 0.01601499 0.01606328 0.01595009 0.01629387 0.01602093
#> rs11206477  0.01522992 0.01486595 0.01490668 0.01480196 0.01512465 0.01484631
#> rs978479    0.01642066 0.01602706 0.01607561 0.01596228 0.01630616 0.01603341
#> rs6684892   0.01738306 0.01696379 0.01701563 0.01689661 0.01726146 0.01696989
#> rs149881092 0.03917473 0.03823671 0.03833955 0.03807307 0.03890204 0.03825441
#> rs2081705   0.01643768 0.01604368 0.01609229 0.01597868 0.01632324 0.01604975
#>                     T7         T8         T9        T10
#> rs6694014   0.01618062 0.01625153 0.01639785 0.01626775
#> rs11206477  0.01499870 0.01506721 0.01522146 0.01508189
#> rs978479    0.01619313 0.01626394 0.01641083 0.01628007
#> rs6684892   0.01713951 0.01721824 0.01737041 0.01723253
#> rs149881092 0.03863522 0.03880163 0.03916327 0.03883610
#> rs2081705   0.01620976 0.01628044 0.01642748 0.01629694
```

In the test dataset there are ten traits with summary association and
standard error data for around 1000 snps. The summary association data
are weakly correlated, owing to generating data from studies containing
overlapping participants (/samples). We can call the correlation and LD
matrices by typing

``` r
trait.cor <- hyprcoloc::test.corr
ld.matrix <- hyprcoloc::test.ld
```

Note that in the sample data, traits 1-5 form a cluster of colocalized
traits; traits 6-8 form a distinct cluster of colocalized traits and;
traits 9-10 form distinct cluster of colocalized traits.

## 2.1 Basic set-up: assuming independence between studies

By default HyPrColoc assumes that each trait is measured in a distinct
study, i.e. that the participants do not overlap between studies, so
that between study estimates of regression coefficients (betas) are
independent. The HyPrColoc function will automatically employ the
Bayesian divisive clustering algorithm to identify clusters of
colocalized traits. Each study (or trait) will be assigned a name
corresponding to their column position and similarly the snps will be
assigned names according to row position. To input trait and snp labels
we make use of the “trait.names” and “snp.id” variables, e.g. 

``` r
traits <- paste0("T", 1:dim(betas)[2]);
rsid <- rownames(betas);
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid);
res;
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3            T9, T10         0.9018             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

From the above output we see that, for each iteration of the algorithm,
HyPrColoc returns:

1)  a cluster of putatively colocalized traits
2)  the posterior probability that these traits are colocalized
3)  the ‘regional association’ probability\* (which is always \> the
    posterior probability)
4)  a candidate causal variant explaining the shared association
5)  the proportion of the posterior probability explained by this
    variant (which represents the HyPrColoc multi-trait fine-mapping
    probability).

\*Note that a ‘large’ regional association probability is evidence that
one or more snps in the region have shared associations across the
traits, the result is similar to a PheWAS (Phenome-wide association
study) and we discuss this more later. The results above show that in
three iterations HyPrColoc identified that traits 1-5 form a cluster of
colocalized traits, traits 6-8 form a separate cluster of colocalized
traits and finally that traits 9 and 10 also colocalize. We see that the
cluster of traits 1-5 have a posterior probability of $1$ of being
colocaized, hence the regional association probability is also $1$ and
that the candidate snp rs11591147 explains 100% of the posterior. Thus,
there is very strong support that traits 1-5 colocalize and that
rs11591147 be taken as the candidate causal snp in the region. On the
other hand, traits 9 and 10 show strong evidence of colocalizing, having
a posterior probability of $0.9$, but there is weak evidence to support
rs7524677 as the causal snp in the region. This example helps to
illustrate that: while a cluster of traits can show strong evidence of
colocalization, the snp most likely to explain colocalization between
the traits can be vague. We assess this in more detail later, when
considering credible sets of snps which explain the colocalization
signal (for each cluster of traits) c.f. section “Computing a credible
set of snps for each cluster of colocalized traits”.

### 2.1.1 Labelling a trait as either continuous or binary in analyses

For technical reasons, analyses have a dependence on whether a trait is
continuous or binary. To let HyPrColoc know which traits are continuous
(coded 0) or binary (coded 1) we use the “binary.outcome” variable. For
example, suppose the first three traits are binary, then we type

``` r
binary.traits = c(1,1,1,rep(0,dim(betas)[2]-3));
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, binary.outcomes = binary.traits);
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3            T9, T10         0.9018             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

### 2.1.2 Choosing a subset of traits to analyze

We can choose to assess evidence of colocalization across a subset of
the traits. For example, let’s suppose interest lies in assessing the
first two traits only, in this situation we make use of the
“trait.subset” variable, e.g.

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, trait.subset = c("T1","T2"));
res
#> $results
#>   iteration traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2              1             1    rs11591147
#>   posterior_explained_by_snp dropped_trait
#> 1                          1            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

## 2.2 Computing a credible set of snps for each cluster of colocalized traits

To output a fine-mapping score, i.e. a probability that a snp is likley
to be causal, for each of the snps in the region we make use of the
“snpscores” variable in analyses. When we do this HyPrColoc outputs a
‘list’ of results where the first element contains the results displayed
previously and the second element containts the snp scores for each of
the clusters of colocalized traits identified. Let’s have a look, if we
now type

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, snpscores = TRUE);
```

then res is a list with two elements: “res$$\[1$$\]” and “res$$\[2$$\]”.
The element “res$$\[1$$\]” contains the results from before, i.e.

``` r
res[[1]];
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3            T9, T10         0.9018             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
```

In the present example, “res$$\[2$$\]” is a list of length 3 containing
the snp scores for each of the 3 clusters of colocalized traits
(presented in the same order as the clusters in “res$$\[1$$\]”), i.e
“res$$\[2$$\]$$\[1$$\]” is a vector and each element of the vector
denotes the probability that a given snp is the causal variant for the
cluster of traits 1-5. “res$$\[2$$\]$$\[2$$\]” returns the same
information but now for the cluster of putatively colocalized traits 6-8
and “res$$\[2$$\]$$\[3$$\]” for the cluster 9-10. For example, the first
few snp scores for cluster 1 (traits 1-5) are:

``` r
head(res[[2]][[1]]);
#>    rs6694014   rs11206477     rs978479    rs6684892  rs149881092    rs2081705 
#> 4.109458e-67 3.004278e-68 5.637121e-67 6.486572e-68 6.063410e-66 5.765875e-67
```

HyPrColoc has an inbuilt function to triage this information down to a
“credible set” of snps for each cluster which uses the output from the
“hyprcoloc” function when “snpsocres = TRUE”. By default, the function
identifies a credible set of snps which explains 95% of the posterior
probability of colocalization for each cluster of colocalized traits. We
this using the “cred.sets” function, i.e.

``` r
cred.sets(res, value = 0.95);
#> [[1]]
#> rs11591147 
#>          1 
#> 
#> [[2]]
#>  rs12117612   rs7532349  rs11206481  rs12145624  rs12724445  rs12126037 
#> 0.419677651 0.419677651 0.025324539 0.022969148 0.022969148 0.022332481 
#>  rs13375783   rs1544909 
#> 0.014513746 0.009991906 
#> 
#> [[3]]
#>  rs7524677  rs7547776  rs7524783  rs6701789  rs1035817  rs7530321  rs7520033 
#> 0.07626197 0.07626197 0.07626197 0.07626197 0.07626197 0.05349125 0.05349125 
#>  rs7553410 rs11206486  rs7524899  rs6681189  rs7537797 rs11206485 rs10888890 
#> 0.05349125 0.02231383 0.02231383 0.02231383 0.02231383 0.02098626 0.02098626 
#>  rs2114574  rs6664660  rs6588539 rs11206489  rs6668051 rs10218512 rs11206492 
#> 0.02098626 0.02098626 0.02098626 0.02098626 0.02098626 0.02098626 0.02098626 
#>  rs4634950  rs7539163  rs7529244  rs6687414  rs7538808 rs11206487 rs12745865 
#> 0.02098626 0.02098626 0.02098626 0.02098626 0.02098626 0.01203687 0.01203687 
#> rs12725873 
#> 0.01203687
```

We see that snp “rs11591147” explains ‘all’ of the posterior probability
of colocalization between the cluster of traits 1-5. This is therefore a
strong candidate causal variant in the region. However, in the second
cluster of traits (6-8), there is no clear candidate causal variante as
both “rs12117612” and “rs7532349” explain equal amounts of the posterior
probability of colocalization and, as we now show, are in perfect LD
with one another:

``` r
ld.matrix["rs12117612","rs7532349"];
#> [1] 1
```

Together these two variants explain nearly 90% of the posterior
probability of colocalization and it is therefore likely\* that the
traits do colocalize, however it is impossible (using HyPrColoc) to
prioritise one of these variants over the other. To prioritise one of
these snps external information would need to be integrated, which can
be (e.g.) biological information or by adding another layer to the
analysis in which the same trait, or traits, are measured in different
studies.

\*One natural question to ask is: if the two snps are in perfect LD why
have we deduced that the traits colocalize? The result is a consequence
of our choice of the prior probability of colocalization. The number of
causal configurations which reflect colocalization between the traits is
much (much!) smaller than configurations which do not represent
colocalization between the traits. This means that if we were to set all
prior configuration probabilities to be equal we would almost certainly
never identify colocalization between the traits, even if the shared
associations between traits were truly owing to a colocalization
mechanism. Thus, a colocalization causal confiugration prior must be
larger than a non-colocalization causal configuration prior for general
(discovery) analyses. But how much larger? There is no simple answer to
this. It is complicated and we therefore dedicate more time to discuss
this now.

# 3 An analysis protocol: assessing stability of clusters via a sensitivity analysis

It is important that users of the software ackowledge that the
performance of HyPrColoc is particularly dependendant on the choice of
prior configuration probabilities used (and any associated
hyper-parameters) as well as the choice of regional and alignment
thresholds, as these combine to quantify a lower bound with which we
accept that a cluster of traits colocalize (i.e. clusters are identified
when $P_RP_{A}\geq P^{\ast}_{R}P^{\ast}_{A}$). In some situations this
senstivity might be modest, whilst in others it might be large. For
example, through extensive simulations, we note that in the analysis of
large numbers of traits, using only the default algorithm settings, can
regularly result in the trait clusters containing (typically only a
single) false positive. Avoiding this issue is complex as it is unlikely
that there exists a one-size-fits-all approach to setting the prior
configuration probabilities and likewise the regional and alignment
threshold parameters. Hence, to go someway to addressing these issues we
provide an analysis protocol template, to assess the strength of any
conclusions.

At the end of this section we introduce a function “sensitivity.plot”
which, on varying the input values for the regional and alignment
thresholds as well as the prior probability of colocalization, returns a
heat-map that helps us to visuale how stable the clusters of colocalized
traits are to variations in the algorithms input parameters. We expect
this function to be a part of standard analyses, helping users to
pinpoint the best candidate clusters of colocalized traits for follow-up
analyses.

## 3.1 Assessing sensitivity to changes in the prior configuration parameters

An important feature of the HyPrColoc software is that it allows for
some sensitivity to the choice of causal configuration priors to be
assessed. Two prior choices are presented: (i) conditionally uniform
priors, which assumes that all causal configurations relating to a given
hypotheses are equally likely, and; (ii) variant specific prioris,
which, for each variant, focuses on tuning the probability that a
variant is colocalized with a subset of traits, for all possible
subsets. HyPrColoc has some dependency to both the type of prior
configuration assumed and the parameters associated with each prior
set-up. In this section we aim to outline an approach to assessing any
sensitivity to these choices and moreover some guidance on drawing
conclusions from sesnsitivity analyses.

HyPrColoc jointly assesses colocalization across multiple traits and we
use this detail to our advantage in our sensitivity analyses. The
general principle is as follows: perform two or more analyses in which
the probability that a variant is associated with more than one trait
decreases and for each assessment compute the number of traits within
each cluster that overlapp between analyses. Any overlap in the
collection of traits within each cluster, which use different choices of
prior paramater values, would then be a ‘stable’ cluster of colocalized
traits. We might wish to do this for each collection of putatively
colocalized traits identified from the primary analyses (which may or
may not use the default settings) or repeat the whole analysis again
under a different parameter specification.

#### 3.1.0.1 Conditionally uniform configuration priors:

The conditionally uniform prior assigns all non-null hypotheses
(i.e. those in which at least one trait has a causal variant in the
region) the same probaility. Note that this does not mean that all
causal configuration prior probailities are the same, rather that the
prior probabilities of each causal configuration associated with a
distinct hypothesis must add up to the same amount. This means that the
conditionally uniform prior benefits from automatically adjusting to
both the number of traits and the number of snps included in analyses.
The approach requires specificying one parameter only, “prior.1”, which
is the ratio of the prior probability of a hypothesis in which at least
one trait has a causal variant in the region $P(H_{j})$ divided by the
null-hypothesis that no traits have a causal variant in the region
$P(H_{0})$, i.e. $prior.1 = \frac{P(H_{j})}{P(H_{0})}$. By default, we
set $prior.1=10^{-4}$, so that a non-null hypothesis is 10000 times less
likely a-priori relative to the null hypothesis. The default choice
$10^{-4}$ is motivated in part by the role of $p1$ in the widely used
sofwtare “COLOC”, where $p1$ is the prior prbability that a snp is
causally associated with one trait in the region whereas $prior.1$ can
be thought of as the prior probability of a hypothesis, in which at
least one trait has a causal variant in the region. As the number of
traits in the sample increases the default choice of $prior.1$ may be
become increasingly inappropriate. It is therefore important to explore
sensitivity of any results to the specification of “prior.1” which we
now do.

As the number of traits in the test dataset is ‘small’, we choose to
repeatedly run HyPrColoc and for each run change the value
$prior.1\in {10^{-4}, 10^{-10}, 10^{-20}, 10^{-25}, 10^{-100}}$. If the
number of traits in the sample were large we might consider selecting a
cluster of traits with which to assess cluster stability.

``` r
prior.options = c(1e-4, 1e-10, 1e-20, 1e-25, 1e-100);
for(i in prior.options){
  res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, 
                   uniform.priors = TRUE, prior.1 = i, reg.steps = 2);
  print(paste0("prior.1 = ",i));
  print(res);
  }
#> [1] "prior.1 = 1e-04"
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9888             1    rs12117612
#> 3         3            T9, T10         0.9799             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-10"
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9888             1    rs12117612
#> 3         3            T9, T10         0.9799             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-20"
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000        1.0000    rs11591147
#> 2         2         T6, T7, T8         0.9887        1.0000    rs12117612
#> 3         3            T9, T10         0.9797        0.9998     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-25"
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5              1         1.000    rs11591147
#> 2         2               None             NA         0.000          <NA>
#> 3         3               None             NA         0.046          <NA>
#> 4         4               None             NA         0.000          <NA>
#> 5         5               None             NA         0.000          <NA>
#>   posterior_explained_by_snp dropped_trait
#> 1                          1          <NA>
#> 2                         NA            T7
#> 3                         NA           T10
#> 4                         NA            T8
#> 5                         NA            T9
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-100"
#> $results
#>   iteration traits posterior_prob regional_prob candidate_snp
#> 1         1   None             NA             0            NA
#> 2         2   None             NA             0            NA
#> 3         3   None             NA             0            NA
#> 4         4   None             NA             0            NA
#> 5         5   None             NA             0            NA
#> 6         6   None             NA             0            NA
#> 7         7   None             NA             0            NA
#> 8         8   None             NA             0            NA
#> 9         9   None             NA             0            NA
#>   posterior_explained_by_snp dropped_trait
#> 1                         NA            T1
#> 2                         NA            T4
#> 3                         NA            T7
#> 4                         NA           T10
#> 5                         NA            T2
#> 6                         NA            T8
#> 7                         NA            T3
#> 8                         NA            T9
#> 9                         NA            T5
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

The results reveal that: in the sample dataset provided and when using
the conditionally uniform priors, the results from HyPrColoc are weakly
dependendent on the specification of “prior.1”. In particular, across
the range $prior.1 \in [10^{-4}, 10^{-20}]$ the HyPrColoc results do not
change beyond the fourth decimal place. We might conclude therefore that
the three clusters of traits 1-5; 6-8 and separately 9-10 are stable
clusters of colocalized traits. When $prior.1 = 10^{-25}$ only the
cluster of traits $1-5$ are deemed to colocalize and when
$prior.1 = 10^{-100}$ no traits are deemed to colocalize. So, do none of
the traits truly colocalize? No, it should be noted that we specified
the values
$prior.1\in {10^{-4}, 10^{-10}, 10^{-20}, 10^{-25}, 10^{-100}}$ not
because these are all ‘reasonable’ values to assess prior sensitivty,
but rather that in the test dataset we tried to find a value,
i.e. $prior.1=10^{-25}$, below which evidence supporting the traits
forming clusters of colocalized traits is ‘small’ so that we fail to
detect colocalization between the traits. In practice and in the absence
of a prior belief concerning $prior.1$, a reasonable choice will depend
on the number of traits under consideration. However, to guide analyses
we suggest comparing results using the default $10^{-4}$ with those when
$prior.1 = 10^{-5}$, i.e. an order of magnitude reduction in “prior.c”,
for sensitivity analyses.

Note that, using the conditionally uniform prior with default
$prior.1=10^{-4}$, the posterior probabilities computed for each cluster
of colocalized traits is larger than those computed using the default
parameters values under the variant specific prior configuration set-up;
we now illustrate this.

#### 3.1.0.2 Variant specific configuration priors (default):

The variant specific causal configuration priors in HyPrColoc are, in
some sense, a multi-trait extention to the causal configuration priors
used in COLOC (for two traits they are identical). Users familiar with
the COLOC software will no doubt know that the performance of COLOC is
sensitive to the choice of the causal configuration prior parameters.
The HyPrColoc multi-trait extention of this prior set-up is also
sensitive to the choice of prior parameters.

The HyPrColoc variant specific priors require the specification of two
parameters: “prior.1” and “prior.c”. The parameter “prior.1” denotes the
probability that a snp is associated with a single trait. “prior.c” is
the conditional colocalization prior. We write $prior.c = 1-prior.2$, so
that: (i) $1-prior.2$ is the prior probaility that a snp is associated
with an additonal trait given that it is associated with one trait; (ii)
$1-(prior.2)^2$ is the prior probability that the snp is associated with
a third trait given it is associated with two other traits; (iii)
$1-(prior.2)^3$ is the prior probability that the snp is associated with
a fourth trait given three and so on… By default $prior.1 = 10^{-4}$,
hence the prior probability that any snp is a ssociated with a single
trait is 1 in 10000 (matching that used in COLOC). The default for
$prior.c = 0.02$, meaning that the prior probability that the snp is
associated with a second trait, conditional on association with one
trait, is 1 in 50; the prior probaility that it is associated with a
third trait given association with two other traits is around 1 in 25;
and so on… Notice that the marginal probability that a snp is associated
with an increasing number of traits is always small and monotonic
decreasing, e.g. $\frac{1}{10^{4}*50*25*\dots}$.

When using the variant specific prior, results tend to be most sensitive
to the choice of $prior.c$ as this controls the prior probability of
each additional colocalized trait. Hence, we focus our sensitivity
analysis on varying this parameter. We consider the range of values
$prior.c \in\{0.05, 0.02, 0.01, 0.005\}$, meaning that the probability
of a snp being associated with a second trait given its association with
one trait can be: (i) 1 in 20, i.e. $prior.c=0.05$; (ii) 1 in 50,
i.e. $prior.c=0.02$; (iii) 1 in 100, i.e. $prior.c=0.01$; (ii) 1 in 200,
i.e. $prior.c=0.005$,

``` r
prior.1 = 1e-4;
prior.c.options = c(0.05, 0.02, 0.01, 0.005);
for(i in prior.c.options){
  res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid,
                   uniform.priors = FALSE, prior.1 = prior.1, prior.c = i);
  print(c(paste0("prior.1 = ",prior.1), paste0("prior.c = ",i)));
  print(res);
}
#> [1] "prior.1 = 1e-04" "prior.c = 0.05" 
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9643             1    rs12117612
#> 3         3            T9, T10         0.9582             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-04" "prior.c = 0.02" 
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3            T9, T10         0.9018             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-04" "prior.c = 0.01" 
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.8464             1    rs12117612
#> 3         3            T9, T10         0.8211             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
#> [1] "prior.1 = 1e-04" "prior.c = 0.005"
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.7342             1    rs12117612
#> 3         3            T9, T10         0.6965             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

The results indicate that traits 1-5 appear to form a ‘stable’ cluster
of colocalized traits, i.e. across the range of prior probabilities
considered the traits 1-5 always colocalize. For the two separate
clusters comprising traits 6-8 and traits 9-10, the results differ
depending on the choice of $prior.c$. Firstly, we note that the
posterior probability of colocalization for each of these clusters
decreases as a function of $prior.c$, but the reduction is not the same
between clusters because the cluster sizes differ. That is, for
$prior.c \in\{0.05, 0.02, 0.01\}$ the posterior probability of the
cluster of size three (traits 6-8) decreases from $0.964$ to $0.846$,
whereas for the cluster of size two (traits 9-10) it reduces from
$0.958$ to $0.821$. In summary, the larger the number of traits in a
cluster the less sensitive results are to the specification of
$prior.c$. We see this as a positive feature of the HyPrColoc variant
specific prior. However, if $prior.c$ is reduced by an order of
magnitude from $10^{-2}$ to $10^{-3}$, i.e. a drop from 1 in 100 to 1 in
1000, we note that of the two clusters of traits (6-8 and 9-10) only
traits 7-8 appear to colocalize and the posterior probability of
colocalization is weaker than before $0.739$. In the absense of a prior
belief concerning $prior.c$, a reasonable value for $prior.c$ will
depend on the number of traits under consideration. As a guide, we
suggested assessing results from $prior.c\in \{0.02, 0.01\}$, i.e. the
HyPrColoc default and when considering that the probability of a snp
having an additional association given an association with one trait is
1 in 100 (our “sensitivity.plot” function will do this for you).

One additional point to consider is evidential strength, if we look at
the output above when setting $prior.c\in \{0.02, 0.01\}$, we note that
the posterior probability of colocalization between the traits in the
clusters 6-8 and 9-10 both fall below $0.85$. Thus, setting a more
stringent threshold for colocalization between the traits, e.g. only
return results when a cluster of traits colocalize with posterior
probability above $0.85$, will necessarily alter the results presented
by HyPrColoc; which we now demonstrate.

#### 3.1.0.3 Evidential strength and the regional and alignment thresholds

HyPrColoc estimates the posterior probability of colocalization between
a cluster of traits by multiplying a regional association probability
$P_{R}$ and an alignment probability $P_{A}$, i.e. $P_{R}*P_{A}$. The
algorithm concludes that a cluster of traits colocalize if these values
are above two threshold values: the regional probability threshold
$P_{R}^{\ast}$ and the alignment probability threshold $P_{A}^{\ast}$.
As noted from the previous section, the posterior probability can vary
by the choice of configuration priors used. Consequently, the default
regional and alignment thresholds vary when using either the
conditionally uniform configuration priors,
i.e. $P_R^{\ast}=P_{A}^{\ast}=0.7$, or the variant specific priors,
i.e. $P_R^{\ast}=P_{A}^{\ast}=0.5$. Accordingly, the algorithm will
deduce that a set of traits are colocalized when $P_RP_{A}\geq 0.49$
using CU priors and $P_RP_{A}\geq 0.25$ when using VS priors. Note,
these parameter choices are a consequence of extensive testing in
simulation scenarios, aiming to maximise the true detection rate whilst
minimising the number of false positives. We therefore DO NOT recommend
reducing the value of either of these parameters. The defaults should be
viewed as lower bounds.

If interest lies in identifying sets of colocalized traits above a
certain posterior probability, more stringent regional and alignment
threshold parameters can be chosen. For example if we set
$P_R^{\ast}=P_{A}^{\ast}=0.95$ so that colocalized traits are identified
when $P_RP_{A}\geq 0.9025$ we see that

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, uniform.priors = FALSE,
                 reg.thresh = 0.95, align.thresh = 0.95);
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000        1.0000    rs11591147
#> 2         2             T7, T8         0.9827        1.0000    rs12117612
#> 3         3               None             NA        1.0000          <NA>
#> 4         4               None             NA        0.9653          <NA>
#>   posterior_explained_by_snp dropped_trait
#> 1                      1.000          <NA>
#> 2                      0.428          <NA>
#> 3                         NA           T10
#> 4                         NA            T9
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

Hence, again we identify that the clusters of traits with very strong
evidence of colocalizing are traits 1-5 and separately 7-8. However,
some of you might notice that in our previous assessment of the data
traits 6-8 colocalized with posterior probability $0.916$ and traits
9-10 with posterior probability $0.902$. Traits 9-10 fall below our
posterior threshold, however traits 6-8 do not. Why is when we set
$P_R^{\ast}=P_{A}^{\ast}=0.95$ do we not identify the full cluster 6-8
rather than just 7-8? The answer is that while the posterior probability
of traits 6-8 colocalizing is $0.916$ either the regional or alignment
probability is not larger than the threshold $0.95$ (when trait 6 is
included in the cluster). In this situation the algorithm will fail to
identify the maximum number of clusters of traits which colocalize above
$0.902$.

In our extensive testing of the HyPrColoc algorithm we noticed that we
can avoid this if we instead set both the regional and alignment
statistics to the minimum posterior probability that determines that a
cluster of traits are colocalized. Hence, in the above example we would
instead set $P_R^{\ast}=P_{A}^{\ast}=0.9025$, i.e. 

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, uniform.priors = FALSE,
                 reg.thresh = 0.9025, align.thresh = 0.9025);
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3               None             NA             1          <NA>
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000          <NA>
#> 2                     0.4197          <NA>
#> 3                         NA           T10
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

which returns the full cluster of traits 6-8.

This results explains why the default values for $P_R^{\ast}$ and
$P_{A}^{\ast}$ are set to be ‘equal’ and ‘both’ reflect the minimum
probability with which we accept clocalization, not the product
$P_{R}*P_{A}$.

#### 3.1.0.4 Analysis protocol: a one stop sensitivity assessment

As discussed above, results from a colocalization analyses might be
sensitive to the choice of: (i) algorithm thresholds,
i.e. colocalization cut-off $=P^{\ast}_{R}*P^{\ast}_{A}$, and; (ii) the
prior belief about traits sharing a causal variant. It is therefore
essential that we explore any sensitivity, particularly if we are aiming
to identify reliable candidate clusters of colocalized traits. HyPrColoc
allows you to do this through a single function “sensitivity.plot”. The
function repeatedly calls “hyprcoloc” for various user defined choices
of the regional and alignment thresholds as well as the colocalization
prior (“prior.c”). The default values for these parameters start from
the algorithms default settings and then increase these so that
identification of clusters of colocalized traits becomes more
challenging (i.e. traits must have stronger evidence of colocalization
in order to be detected). For example

``` r
sensitivity.plot(betas, ses, trait.names = traits, snp.id=rsid, 
                 reg.thresh = c(0.6,0.7,0.8,0.9), align.thresh = c(0.6,0.7,0.8,0.9), prior.c = c(0.02, 0.01, 0.005), equal.thresholds = FALSE);
```

![](hyprcoloc_files/figure-gfm/unnamed-chunk-18-1.png)<!-- -->

``` r
# Or by fixing the regional and aligment thresholds to be equal
# sensitivity.plot(betas, ses, trait.names = traits, snp.id=rsid, 
#                  reg.thresh = c(0.6,0.7,0.8,0.9), prior.c = c(0.02, 0.01, 0.005), equal.thresholds = TRUE);
```

The output is a heat-map which helps us to visualise the clusters of
colocalised traits. Traits which cluster across all values considered
are given a score of 1 and traits which never cluster are given a score
of 0; traits which ocassionally cluster have a score between 0 and 1.
Hence, our confidence in a cluster increases with increasing score -
appearing as a darker block of clustered traits in the heat-map. The
heat-map reflects what we have already seen from our individual
analyses: traits 1-5 form a very strong cluster of colocalized traits;
traits 6-8 form a second cluster, however for certain values of the
input parameters trait 6 is dropped from this cluster and; traits 9-10
form a thrid cluster, however identification of this cluster is
difficult for more stringent values of the thresholds and the prior
probability of colocalization. It is clear from our figure that there
are three distinct clusters of colocalized traits in our sample,
matching the true data generating mechanism. However, our most confident
clusters of colocalised traits are traits 1-5 and separately traits 7-8.
Although we have set the benchmark quite high, the two clusters 6-8 and
9-10 are present in around 70% of the (sensitivity) scenarios considered
making them reasonable candidates also. To see this we can require that
“sensitivity.plot” returns the similarity matrix used to plot the
heat-map:

``` r
res = sensitivity.plot(betas, ses, trait.names = traits, snp.id=rsid, 
                 reg.thresh = c(0.6,0.7,0.8,0.9), align.thresh = c(0.6,0.7,0.8,0.9), prior.c = c(0.02, 0.01, 0.005), equal.thresholds = FALSE, similarity.matrix = TRUE);
```

![](hyprcoloc_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

``` r
# heatmap is found by typing 
heatmap.plot = res[[1]]; # (plotted previously)
# similarity matrix is found by typing
sim.mat = res[[2]];
sim.mat;
#>     T1 T2 T3 T4 T5   T6   T7   T8        T9       T10
#> T1   1  1  1  1  1 0.00 0.00 0.00 0.0000000 0.0000000
#> T2   1  1  1  1  1 0.00 0.00 0.00 0.0000000 0.0000000
#> T3   1  1  1  1  1 0.00 0.00 0.00 0.0000000 0.0000000
#> T4   1  1  1  1  1 0.00 0.00 0.00 0.0000000 0.0000000
#> T5   1  1  1  1  1 0.00 0.00 0.00 0.0000000 0.0000000
#> T6   0  0  0  0  0 1.00 0.75 0.75 0.0000000 0.0000000
#> T7   0  0  0  0  0 0.75 1.00 1.00 0.0000000 0.0000000
#> T8   0  0  0  0  0 0.75 1.00 1.00 0.0000000 0.0000000
#> T9   0  0  0  0  0 0.00 0.00 0.00 1.0000000 0.6666667
#> T10  0  0  0  0  0 0.00 0.00 0.00 0.6666667 1.0000000
```

We se that in 75% of scenarios traits 6-8 form a cluserter of
colocalized traits and in around 67% of scenarios traits 9-10 form a
cluster of colocalized traits.

## 3.2 The Bayesian divisive clustering algorithm

The Bayesian divisive clustering algorithm identifies clusters of
colocalized traits in the sample. The algorithm is automatically used
when all traits are identified as not sharing a causal variant. For each
iteration of the algorithm, the goal is to identify a subset of traits
with the greatest evidence of colocalization. HyPrColoc does this using
one of two (user defined) approaches: (i) the regional or (ii) alignment
selection. The regional selection criterion is computed from a
collection of hypotheses which assume that all traits do not colocalize
because one of the traits does not have a causal variant in the region.
The alignment selection criterion, however, is computed from hypotheses
which assume that all traits do not colocalize because one of the traits
has a causal variant elsewhere in the region. The alignment selection
process searches a much larger amount of the causal configuration space
and is thus more computationally expensive (but potentially more robust)
than regional selection. Typically, both strategies perform similarly
and we therefore chose the regional selection as the default setting.

### 3.2.1 Assessing differences between the Bayesian divisive clustering criteria

The regional selection criterion:

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, bb.selection = "regional");
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3            T9, T10         0.9018             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

The alignment selection criterion:

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, bb.selection = "align");
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9164             1    rs12117612
#> 3         3            T9, T10         0.9018             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.4197            NA
#> 3                     0.0763            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

Hence, in our test dataset there is no difference in the results when
using either the reginoal or the alignment selection criterion.

### 3.2.2 Switching the algorithm off

We can choose to switch the Bayesian divisive clustering algorithm off
and assess whether all traits colocalize.

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, bb.alg = FALSE);
res
#> $results
#>                                    traits posterior_prob regional_prob
#> 1 T1, T2, T3, T4, T5, T6, T7, T8, T9, T10              0        0.0132
#>   candidate_snp posterior_explained_by_snp
#> 1    rs11591147                          1
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

# 4 Mapping pleiotropy: an alternative to PheWAS

The regional association can be used as an alternative to a PheWAS
assessment. To see this, we first discuss how the regional association
probability is computed.

For each of $Q$ snps in the region, we compute the probability that a
snp is associated with all traits and divide this by the sum of the
probabilities that: (i) one of the $Q$ snps is associated with all
traits; (ii) one of the snps is associated with all but one trait; (iii)
one of the snps is associated with all but two traits and so on… Notably
there is no restriction on the number of causal variants in the region
under a regional association assessment. A ‘large’ (close to 1) regional
association probability indicates that one or more snps in the region
share an association with all traits. This may be owing to one or more
shared causal variants between the traits or because all traits have one
or more causal variants and at least one causal variant per trait is in
strong LD with a causal variant from each of the other traits. Small
values indicate that there is not a snp in the region associated with
all traits, this does not preclude the possibly that one or more snps
might be associated with a subset of the traits, however. The divisive
clustering algorithm contains the option (bb.selection = “reg.only”) to
identify clusters of traits which share a regional association only,
avoiding the alignment phase and therefore the colocalization
assessment. This function therefore allows us to assess evidence of
plieotropy between subsets (i.e. clusters) of traits.

Clusters of regionally associated traits are identified when the
regional association probability is above a user defined threshold
(default $P_{R}^{\ast}= 0.5$, which is likely to be anti-conservative
and therefore should be increased). As an example, below we assess any
evidence of regional associations between the traits in the sample
dataset and we set the threshold regional association probability to be
$0.9$:

``` r
ptm = proc.time();
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, bb.selection = "reg.only", reg.thresh = 0.9);
proc.time() - ptm;
#> 使用者   系統   流逝 
#>   0.01   0.00   0.02
res
#> $results
#>   iteration              traits posterior_prob regional_prob candidate_snp
#> 1         1  T1, T2, T3, T4, T5              0        1.0000    rs11591147
#> 2         2 T6, T7, T8, T9, T10              0        0.9473    rs12117612
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.3137            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

The results indicate that there exists one or more snps that share
associations with traits 1-5, a result anticipated from the results of
our previous colocalization analysis. However, by avoiding a
colocalization assessment we identify that traits 6-10 all appear to
have a shared association with at least one snp, the most likely shared
snp is rs12117612 which explains 31.4% of the regional association
probability. This contrasts with the results from our earlier
colocalization assessment which indicated that traits 6-10 do not all
colocalize and highlights some distinction between traits which have a
shared genetic associations (PheWAS) and those which colocalize at a
single causal variant.

As a final point of note, the regional association statistic is computed
from a much smaller number of causal configurations than a full
colocalization assessment. The upshot is that the Bayesian divisive
clustering algorithm, using only the regional statistic, can rapidly
identify clusters of traits which shared genetic associations amongst
very large numbers of traits. As an example, try employing a regional
only assessment to the dataset of 1000 traits that we build later in the
vignette. We can also assess any sensitivity in our results to the
choice of regional threshold using the “sensitivity.plot” function,
which we introduce in the next section.

# 5 Analysing large numbers of traits

A common question we are asked is: can HyPrColoc jointly and quickly
analyse hundreds or thousands of traits? As we have seen from the
previous section, assessing the stability of clusters of putatively
colocalized traits is no simple task and this problem can become more
difficult as the number of traits under consideration increases. The
approach to assessing sensitivity of results to the choice of prior and
parameters, as well as evidential strength, is the same whether we
analyse 10 traits or 1000. Given this, here we focus on demonstrating
the ability of HyPrColoc to identify clusters of colocalized traits from
100 then 1000 traits. We do this by replicating the test dataset of 10
traits at first 10 times: to build a dataset of 100 traits in which
traits 1-50 form a cluster of colocalized traits, traits 51-80 form a
second cluster of colocalized traits and finally traits 81-100 form a
third cluster of colocalized traits, e.g.

``` r
# 100 traits
# beta100 replicates the betas for clusters 1, 2 and 3 10 times  
traits100 <- paste0("T", 1:100)
betas100 = ses100 = NULL;
clusters = c(1,2,3);
clst.traits = list(1:5, 6:8, 9:10);
times = 10;
  for(c in clusters){
    tmp.betas = tmp.ses = NULL;
    traits = clst.traits[[c]];
        for(i in 1:times){
            tmp.betas = cbind(tmp.betas, betas[,traits]); 
            tmp.ses = cbind(tmp.ses, ses[,traits]);
          }
    betas100 = cbind(betas100, tmp.betas);
    ses100 = cbind(ses100, tmp.ses)
  }
ptm = proc.time(); 
res = hyprcoloc(betas100, ses100, trait.names = traits100, snp.id = rsid);
proc.time()-ptm;
#> 使用者   系統   流逝 
#>   0.33   0.08   0.41

# print the number of traits in each cluster
clusters = which(! is.na(res$results$traits));
for(c in clusters){
traits = unlist(strsplit(res$results$traits[c], split=", "));
head.traits = head(traits);
tail.traits = tail(traits);
# print the head and tail of the traits in each cluster
print(paste0("cluster ",c," contains ",length(traits), " traits"));
print("#####");
print(c(head.traits, "...", tail.traits));
print("#####");
}
#> [1] "cluster 1 contains 50 traits"
#> [1] "#####"
#>  [1] "T1"  "T2"  "T3"  "T4"  "T5"  "T6"  "..." "T45" "T46" "T47" "T48" "T49"
#> [13] "T50"
#> [1] "#####"
#> [1] "cluster 2 contains 30 traits"
#> [1] "#####"
#>  [1] "T51" "T52" "T53" "T54" "T55" "T56" "..." "T75" "T76" "T77" "T78" "T79"
#> [13] "T80"
#> [1] "#####"
#> [1] "cluster 3 contains 20 traits"
#> [1] "#####"
#>  [1] "T81"  "T82"  "T83"  "T84"  "T85"  "T86"  "..."  "T95"  "T96"  "T97" 
#> [11] "T98"  "T99"  "T100"
#> [1] "#####"
```

There are several points to note: (i) HyPrColoc correctly identified the
three clusters of colocalized traits; (ii) by borrowing strength across
more (colocalized) traits, the posterior probability explained by each
of the candidate snps has increased meaning we have incresed confidence
that we have identified the underlying causal variant (or one in perfect
LD with it) and; (iii) HyPrColoc did this in under 1 second. Let’s do
the same thing for 1000 traits,

``` r
# 1000 traits
traits1000 <- paste0("T", 1:1000)
betas1000 = ses1000 = NULL;
clusters = c(1,2,3);
clst.traits = list(1:5, 6:8, 9:10);
times = 100;
  for(c in clusters){
    tmp.betas = tmp.ses = NULL;
    traits = clst.traits[[c]];
        for(i in 1:times){
            tmp.betas = cbind(tmp.betas, betas[,traits]); 
            tmp.ses = cbind(tmp.ses, ses[,traits]);
          }
    betas1000 = cbind(betas1000, tmp.betas);
    ses1000 = cbind(ses1000, tmp.ses)
  }
ptm = proc.time(); 
res = hyprcoloc(betas1000, ses1000, trait.names = traits1000, snp.id = rsid);
proc.time()-ptm;
#> 使用者   系統   流逝 
#>  25.22   7.17  32.45

# print the number of traits in each cluster
# print the number of traits in each cluster
clusters = which(! is.na(res$results$traits));
for(c in clusters){
traits = unlist(strsplit(res$results$traits[c], split=", "));
head.traits = head(traits);
tail.traits = tail(traits);
# print the head and tail of the traits in each cluster
print(paste0("cluster ",c," contains ",length(traits), " traits"));
print("#####");
print(c(head.traits, "...", tail.traits));
print("#####");
}
#> [1] "cluster 1 contains 500 traits"
#> [1] "#####"
#>  [1] "T1"   "T2"   "T3"   "T4"   "T5"   "T6"   "..."  "T495" "T496" "T497"
#> [11] "T498" "T499" "T500"
#> [1] "#####"
#> [1] "cluster 2 contains 300 traits"
#> [1] "#####"
#>  [1] "T501" "T502" "T503" "T504" "T505" "T506" "..."  "T795" "T796" "T797"
#> [11] "T798" "T799" "T800"
#> [1] "#####"
#> [1] "cluster 3 contains 200 traits"
#> [1] "#####"
#>  [1] "T801"  "T802"  "T803"  "T804"  "T805"  "T806"  "..."   "T995"  "T996" 
#> [10] "T997"  "T998"  "T999"  "T1000"
#> [1] "#####"
```

Again, for 1000 traits HyPrColoc correctly identified the three clusters
of colocalized traits. This time, however, it took longer to do so
(still under 1 minute). The length of time the algorithm takes depends
on the complexity of the clustering between the traits. It is therefore
very much dependent on the region and traits under consideration.

Notice that the posterior probability of colocalization has dropped
quite a lot for clusters 2 and 3 when analysing 1000 traits. This is a
feature of assessing large numbers of traits jointly in a Bayesian
framework, we discuss this nuance in more detail in the HyPrColoc text.
As a final point of note, we notice that the posterior probability
explained by the candidate snps in clusters 2 and 3 is ‘maximized’. That
is, we know from our previous assessment of the credible sets of snps,
in both clusters, that the candidate snp for cluster 2 is in perfect LD
with one other snp. In cluster 3, the candidate snp is in perfect LD
with four other snps. Hence, the maximum evidential support for a single
causal snp in clusters 2 and 3 can be 50% and 20% respectively, which is
what we identify in our analysis.

# 6 Analysing correlated traits

When interest lies in identifying colocalization across a set of traits
whose summary data are correlated (due to overlapping participants)
three additional pieces of information are required: (i) a matrix
containing the putative correlation between the traits; (ii) a matrix
containing the LD between the snps in the region and; (iii) a matrix
containing the proportion of shared participants between each pair of
studies. Note the matrix of overlapping sample proportions has diagonal
elements set to 1 and off diagonal elements $(i,j)$ denote the
proportion of overlap between the $i$’th and $j$’th studies, i.e. if N
denotes the total number of overlapping participants and
$\{N_{i},N_{j}\}$ denote the numbers of participants in the $i$’th and
$j$’th studies respectively, then $N/\left(N_{i}N_{j}\right)$ is the
$(i,j)=(j,i)$ element. As sample overlap information may not always be
available, HyPrColoc defaults to assumes all studies have the same
participants.

### 6.0.1 Important, before going further…

Question: should analyses be adjusted for observed correlation between
study summary data?

It is our recommendation that users first consider whether adjusting for
observed correlation between summary data is necessary. We here detail
several points to consider.

Traits which are strongly correlated are in general more likely to
colocalize. This has two important consequences: (i) whilst special
exceptions exist, generally the a-priori probability of colocalization
between two strongly correlated traits is necessarily larger than two
traits that are weakly correlated, analyses which ignore this are less
likely to capture the largest set of traits which are truley colocalized
(see the HyPrColoc text for more on this) and consequently; (ii) if a
subset of the cluster of traits which truley colocalize are identified,
it is possible (moreover likely) that one or more of the correlated
traits has been wrongly removed from the cluster due to mathematical
subtleties when inverting the correlation matrix (to estimate the
posterior probability of colocalization), and not because of some
underlying biological importance that one of the traits has over the
another.

As something of a discussion point and an example of (ii), let’s suppose
interest lies in assessing diastolic blood pressure (DBP) and systolic
blood pressure (SBP), two strongly correlated traits, and coronary heart
disease. Suppose we perform a joint analysis of the three traits which
accounts for correlation in the summary data, but not a-priori trait
correlation. Suppose further that we identify a region in which SBP and
CHD colocalize only. What do we conclude? Do we assume that our findings
provide evidence toward SBP being the more important blood pressure
trait for CHD risk in this region? Now suppose we question this result
by performing two separate colocalization analyses (i) including SBP and
CHD only and; (ii) including DBP and CHD only. Our findings reveal that
in this region CHD colocalizes with SBP and separately colocalizes with
DBP at the same snp. What then is the conclusion? Is there a more
important blood pressure trait here? Well, we demonstrate in the
HyPrColoc supplementary material that if analyses account for ‘known’,
i.e. a-priori, trait correlation by upweighting the prior probability of
colocalization between strongly correlated traits as well as accounting
for observed correlation, through the summary effects, HyPrColoc
regularly identifies the full set of truly colocalized traits. So, in
the case of SBP, DBP and CHD, additionally accounting for ‘known’
correlation between SBP and DBP might overcome this (hypothetical)
issue. The difficulty with this approach is identifying a robust way to
incorporate ‘known’ trait correlation into analyses. This is an
outstanding problem. It appears it can be avoided however, by ignoring
both observed and a-priori correlations in analyses, i.e. assuming
independence between the studies and the traits. We demonstrate that
this strategy regularly identifies the maximum number of truly
colocalized traits and at a fraction of the computational cost of
analyses which account for correlation.

### 6.0.2 Analysing correlated traits, continued…

Our command line tool can automatically compute trait correlation (using
a tetrachoric correlation technique) and LD information. However, the
R-package requires users to have this information to hand.

``` r
# default assumption, presented for clarity:
sample.overlap = matrix(1, dim(betas)[2], dim(betas)[2]); 
```

#### 6.0.2.1 Variant specific priors

``` r
traits <- paste0("T", 1:dim(betas)[2]);
ptm = proc.time();
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, trait.cor = trait.cor, 
                 ld.matrix = ld.matrix, sample.overlap = sample.overlap, uniform.priors = FALSE);
time.corr = proc.time() - ptm;
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.8257             1    rs12117612
#> 3         3            T9, T10         0.8944             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.3535            NA
#> 3                     0.0696            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

#### 6.0.2.2 Conditionally uniform priors

``` r
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid, trait.cor = trait.cor,
                 ld.matrix = ld.matrix, sample.overlap = sample.overlap, uniform.priors = TRUE);
res
#> $results
#>   iteration             traits posterior_prob regional_prob candidate_snp
#> 1         1 T1, T2, T3, T4, T5         1.0000             1    rs11591147
#> 2         2         T6, T7, T8         0.9744             1    rs12117612
#> 3         3            T9, T10         0.9782             1     rs7524677
#>   posterior_explained_by_snp dropped_trait
#> 1                     1.0000            NA
#> 2                     0.3535            NA
#> 3                     0.0696            NA
#> 
#> attr(,"class")
#> [1] "hyprcoloc"
```

#### 6.0.2.3 Assuming independence (which correctly captures the data generating model)

Recall that when we incorrectly assumed that the traits were measured in
studies with no overlapping participants, we correctly detected that
traits 1-5; traits 6-8 and traits 9-10 form three clusters of
colocalized traits and moreover we identified this at a fraction of the
computational cost of accounting for correlations between summary data

``` r
ptm = proc.time();
res <- hyprcoloc(betas, ses, trait.names=traits, snp.id=rsid);
time.ind = proc.time() - ptm;
time.ind;
#> 使用者   系統   流逝 
#>   0.00   0.01   0.01
time.corr;
#> 使用者   系統   流逝 
#>  33.02   0.37  33.40
```

Thus, assuming independence between the studies not only correctly
identifies the three clusters of colocalized traits, but does so around
3500 times faster than accounting for non-indepdence.
