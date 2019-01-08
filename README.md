# Introduction

Fetal DNA fraction is an important estimate during non-invasive prenatal testing (NIPT). In routine practice, this figure is ideally derived from shallow-depth whole-genome sequencing copy number data, preventing the need for additional experimental assays. The fraction of mapped reads against chromosome Y enables proper quantification, however, it’s use is limited to male feti. For pregnancies of female feti, predictive models are required. Regardless, an accurate software to efficiently train these is currently non-existent. As a result, laboratories are often restricted to pre-trained alternatives during routine testing, which introduces bias due to differences in laboratory and in silico procedures. Here we present PREFACE (PREdict FetAl ComponEnt), a novel bioinformatics pipeline that combines the strengths of principal component analysis and neural networks. For sets of roughly 1100 male NIPT samples, we report a cross-validated Pearson correlation of 0.9 between predictions and fetal fractions according to Y chromosomal read counts. Our approach enables training with both male and unsupervised female feti. Using all available cases (n<sub>female</sub>=2468, n<sub>male</sub>=2723), the correlation metric reached 0.94. In addition, PREFACE provides the fetal fraction based on the copy number state of the X chromosome. In combination, these measures can predict mixed multiple pregnancies, sex chromosomal aneuploidies and the source of observed aberrations. To date, no other shallow whole-genome sequencing based approach has been reported to perform better for predicting fetal fraction. For details please consult the [paper](publication).   

# Manual

## Required files

### Copy number alteration .bed files

Each sample (whether it is used for training or for predicting) should be passed to PREFACE in the following format. During benchmarking, copy number normalization was executed by [WisecondorX](https://github.com/CenterForMedicalGeneticsGhent/WisecondorX/), yet PREFACE is not limited to any copy number alteration software. Note that the default output of WisecondorX is directly interpretable by PREFACE.  

- Example: ```./examples/infile.bed```  
- Tab-separated file with at least four columns.  
- The name of these columns (passed as a header) must be 'chr', 'start', 'end' and 'ratio'.  
    - The possible values for 'chr' are 1 until 22 and X and Y (capital letters).  
    - 'ratio' corresponds to the log2-transformed ratio between the observed and expected copy number.  
    - Loci can be indeterminable (e.g. at repeats). Here, 'ratio' values should be expressed as 'NaN'.  
- The order of rows does not matter. Yet, it is of paramount importance that, for a certain line, file X deals with the same locus as file Y. This implies, of course, that all copy number alteration files have the same number of lines.  
- PREFACE was validated using a bin size of 100 kb, lower bin sizes are expected to perform at least as well.  

### PREFACE's config.txt

For training, PREFACE requires a config file which contains every training case.   

- Example: ```./examples/config.txt```  
- Tab-separated file with at least four columns.  
- The name of these columns (passed as a header) must be 'ID', 'filepath', 'gender' and 'FFY'.  
    - 'ID': a unique identifier should be given to each of the samples.  
    - 'filepath': the full absolute path of the copy number alteration files.  
    - 'gender': either 'M' (male) or 'F' (female), representing fetal gender. Twins/triplets/...  might be included, yet they can only be labeled as 'M' if they are all male; if not, they should be 'F'.  
    - 'FFY': the fetal fraction according to the number of mapped Y-reads. One can use any formula he/she believes performs best at quantifying the actual fetal fraction. This measurement will be ignored for female feti.  

## Model training

```bash

RScript PREFACE.R train --config path/to/config.txt --outdir path/to/dir/ [optional arguments]  
```

<br>Optional argument <br><br> | Function  
:--- | :---  
`--nfeat x` | Number of principal components to use during modeling (default: x=50)  
`--cpus x` | Use for multiprocessing, number of requested threads (default: x=1)  
`--olm` | It might be possible the neural network does not converge; or for your kind of data/sample size, an ordinary linear model might be a better option. In these cases, use this flag.  
`--noskewcorrect` | This flag ascertains the best fit for most (instead of all) of the data is generated.  

## Predicting

```bash

RScript PREFACE.R predict --infile path/to/infile.bed --model path/to/model.RData [optional arguments]  
```

<br>Optional argument <br><br> | Function  
:--- | :---  
`--json x` | Predictions are written to the stdout. Use this flag for json format. Optionally provide 'x' to generate .json file x.  

## Remarks

- The most important parameter is `--nfeat`
    - It represents the number of principal components that will be used as features during model training. Depending on the used copy number alteration software, bin size and number of training samples, it might have different optimal values. In general, I would recommend to train a model using the default parameters. The output will contain a plot that enables you to review the selected `--nfeat`. Two parts should be seen in the proportion of variance across the principal components (indexed in order of importance):
        - A 'random' phase (representing PCs that explain variance caused by, inter alia, fetal fraction).
        - A 'non-random' phase (representing PCs that explain variance caused by naturally occurring Gaussian noise).
    - An optimal `--nfeat` captures the 'random' phase (as shown in the example at `./examples/overall_performance.png`). Capturing too much of the 'non-random phase' could lead to convergence problems during modeling.
    - If you are not satisfied with the concordance of your model or with the position of `--nfeat`, re-run with a different number of features.  
- Note that the resulting model (the one that is written to the output) will probably be a bit more accurate than what is claimed by the statistics file and figures. This is because PREFACE uses a cross-validation strategy where 10% of the male samples are excluded from training, after which these 10% serve as validation cases. This process is repeated 10 times. Therefore, the final performance measurements are based on models trained with only 90% of the male feti, yet the resulting model is trained with all provided cases.  

# Required R packages

- doParallel (v1.0.14)  
- foreach (v1.4.4)  
- neuralnet (v1.33)  
- glmnet (v2.0-16)  
- data.table (v1.11.8)  
- MASS (v7.3-49)  


Other versions are of course expected to work equally well. To install within R use  

```bash

install.packages(c('data.table', 'glmnet', 'neuralnet', 'foreach', 'doParallel', 'MASS'))
```