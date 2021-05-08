# TargetFinder

TargetFinder is a pipeline for identifying and characterizing the gene targets of distal enhancers.  In "Enhancer-promoter interactions are encoded by complex genomic signatures on looping chromatin", Nature Genetics 2015 (to appear), we used TargetFinder to:

- Show that the three-dimensional structure of the genome is encoded via relatively few one-dimensional assays including protein binding and epigenetic modifications.
- Learn complex signatures of looping interactions, showing that marks on looping chromatin (the *window* between an enhancer and promoter) are more predictive than enhancer- or promoter-proximal marks. A subset of marks flanking the enhancer (+/- 3kb) recover much of the predictive power of the full window.
- Determine minimal predictive subsets of features across 6 ENCODE cell lines and one "combined" line, identifying highly predictive lineage-specific datasets as well as those likely to generalize to new cell lines.
- Quickly screen new functional genomics datasets for relevance to looping interactions.
- Identify under-studied proteins and epigenetic modifications that interact with known looping factors to greatly reduce false positive predictions.

Much of TargetFinder's performance is due capturing complex interactions between diverse datasets, most of which are derived from ChIP-seq assays. Unfortunately, due to the high variability of ChIP-seq, you currently *cannot* use TargetFinder to:

- Learn a model from cell lines where high-resolution interaction data exists (e.g. Rao et al. 2014), and make predictions on a new cell line lacking such interaction data.

This was one of the original goals of TargetFinder, and our current work is focused on normalization methods to make such applications possible.  However, you *can* use TargetFinder to identify enhancer-promoter interactions or gain mechanistic insights into looping interactions for cell lines where high-resolution interaction data is available.

## Pre-Generated Training Data

Labeled training datasets used in the paper are available in the `paper/targetfinder` directory, where each cell line has its own subdirectory. Furthermore, each cell line has 3 training datasets with their own subdirectories: one with features generated for the enhancer and promoter only (EP), one for promoters and extended enhancers (EEP), and one for promoters, enhancers, and the window between (EPW). For example, `paper/targetfinder/HeLa-S3/output-eep` contains training data for the HeLa-S3 cell line using promoter and extended enhancer features.

Within each output directory you will find most of the following files, depending on dataset availability:

Filename | Description  
--- | ---  
`training.csv.gz` | Labeled training data in compressed CSV format, slower to load but compatible with Python and R.  
`training.h5` | Labeled training data in HDF5 format, fast to load via Python but incompatible with R.  
`predictions-baseline.csv` | Baseline predictions generated by `DummyClassifier`.  
`predictions-gbm.csv` | Predictions generated by `GradientBoostingClassifier`.  
`predictions-linear.csv` | Predictions generated by `LinearSVC`.  
`predictions-tree.csv` | Predictions generated by `DecisionTreeClassifier`.  
`rfe-gbm.csv` | Performance of Recursive Feature Elimination with `GradientBoostingClassifier`.  
`feature_importances-gbm.csv` | Feature importances calculated by `GradientBoostingClassifier`, averaged over 10 models using different random seeds. Contains columns for the mean, standard deviation, and standard error.  
`enhancers.bed` | Genomic coordinates of enhancers from ENCODE or Roadmap Epigenomics in BED format.  
`promoters.bed` | Genomic coordinates of promoters from ENCODE or Roadmap Epigenomics in BED format. Must be expressed with mean RPKM > 0.3 and IDR < 0.1.  
`tss.bed` | Genomic coordinates of Transcription Start Sites from ENCODE or Roadmap Epigenomics in BED format.  
`pairs.csv` | Genomic coordinates for interacting and distance-matched non-interacting enhancer-promoter pairs, as well as other columns useful for analysis.  
`cage.bed.gz` | Processed CAGE data from ENCODE in compressed BED format.  
`methylation.bed.gz` | Processed Methyl-RRBS data from ENCODE in compressed BED format.  
`peaks.bed.gz` | Processed ChIP-seq, DNase-seq, and FAIRE-seq data from ENCODE and others in compressed BED format.  

Predictions are generated during 10-fold cross-validation, and the prediction CSVs are obtained by concatenating predictions from each of the 10 test sets.

## Custom Analyses

Below we give a very simple example of loading the pre-generated training data, dropping non-predictors, running 10-fold cross-validation with gradient boosting, printing the average performance, and printing the top 16 predictive features using `scikit-learn`.

```python
import pandas as pd

from sklearn.cross_validation import StratifiedKFold, cross_val_score
from sklearn.ensemble import GradientBoostingClassifier

nonpredictors = ['enhancer_chrom', 'enhancer_start', 'enhancer_end', 'promoter_chrom', 'promoter_start', 'promoter_end', 'window_chrom', 'window_start', 'window_end', 'window_name', 'active_promoters_in_window', 'interactions_in_window', 'enhancer_distance_to_promoter', 'bin', 'label']

training_df = pd.read_hdf('paper/targetfinder/K562/output-epw/training.h5', 'training').set_index(['enhancer_name', 'promoter_name'])
predictors_df = training_df.drop(nonpredictors, axis = 1)
labels = training_df['label']

estimator = GradientBoostingClassifier(n_estimators = 4000, learning_rate = 0.1, max_depth = 5, max_features = 'log2', random_state = 0)
cv = StratifiedKFold(y = labels, n_folds = 10, shuffle = True, random_state = 0)

scores = cross_val_score(estimator, predictors_df, labels, scoring = 'f1', cv = cv, n_jobs = -1)
print('{:2f} {:2f}'.format(scores.mean(), scores.std()))

estimator.fit(predictors_df, labels)
importances = pd.Series(estimator.feature_importances_, index = predictors_df.columns).sort_values(ascending = False)
print(importances.head(16))
```

This will run one cross-validation fold per CPU core, but will still take a substantial amount of time given the size of the dataset and the computational complexity of boosting.

The above code is a flexible starting point. One could substitute `GradientBoostingClassifier` for `DecisionTreeClassifier` to compare performance with a single decision tree, for example (given the correct imports).

We repeated most of our analyses in R using `caret` and `gbm`, though this is not recommended due to the slowness of R. The boosting code in `scikit-learn` is substantially different from and has features not available in `gbm`, but the results should roughly be the same. Simply load `training.csv.gz` instead of `training.h5` in your R code to get started. Hadley's `read_csv` function in the `readr` package will save substantial loading time.

## Re-Generating Training Data

We recommend using pre-generated files if you wish to run custom analyses. However, if you have multiple CPU cores and large amounts of RAM (48+ GB), you can re-generate these files in about an hour. It is possible to use less RAM at the cost of slower generation time -- email me.

TargetFinder requires:

- Python (tested 3.5.1)
- scikit-learn (tested 0.17)
- pandas (tested 0.17.1)
- numexpr (tested 2.4.6)
- pytables (tested 3.2.2)
- bedtools (tested 2.25.0)

We recommend the [Anaconda](https://www.continuum.io) Python distribution, and specifically the minimal [Miniconda](http://conda.pydata.org/miniconda.html) installer. After running the installer and adding Miniconda to your `$PATH` variable, simply run

    conda install scikit-learn pandas numexpr pytables

Bedtools 2.25.0 or later must also be available in your `$PATH`. If you use [Homebrew](http://brew.sh) for OS X, `brew tap homebrew/science; brew install bedtools` will (as of 2/22/16) install 2.24.0 which is lacking a flag necessary for TargetFinder to run.

To generate fresh training data for a particular cell line and dataset (e.g. K562 with EPW features), run:

	./generate_region.sh K562/epw.json

from the repo directory.  This will generate enhancers, promoters, enhancer-promoter pairs, and features for those pairs using the JSON configuration file for that cell line and dataset. The resulting `training.h5` file can be converted to a compressed CSV using the bundled `utils/hdf_to_csv.py` script if desired. Either of the resulting files should be equivalent (modulo random number generation) to pre-generated training datasets in the repository.

## Configuration Files

Each cell line and dataset (EP, EEP, and EPW) have a JSON configuration file.  These are simply key-value pairs in a human-readable format similar to a Python dictionary, and are simple to load in R or Python if desired. For example, the `K562/ep.json` file consists of the following:

```python
{
  "working_dir": "output-ep",
  "training_fn": "training.h5",
  "dependent_variable": "label",
  "dependent_values": [0, 1],
  "sample_name_variables": ["enhancer_name", "promoter_name"],
  "nonpredictor_variables": ["enhancer_chrom", "enhancer_start", "enhancer_end", "promoter_chrom", "promoter_start", "promoter_end", "window_chrom", "window_start", "window_end", "window_name", "active_promoters_in_window", "interactions_in_window", "enhancer_distance_to_promoter", "bin"],
  "regions": ["enhancer", "promoter"],
  "enhancer_extension_size": 0,
  "promoter_extension_size": 0
}
```

This specifies a working directory (relative to the configuration file) where generated files should be placed, the desired name of the training file, the column name of the dependent variable (labels) and what values it takes, column names that should be used as the index (row names) of the training data once loaded, column names that should be dropped before fitting a classifier, what genomic regions to generate features for, and how far to extend enhancers and promoters (in bp).

Since most settings are shared between datasets within a given cell line, we chose a single file (`ep.json`) to be the primary configuration from which other configurations inherit. Thus we can define a new configuration (`epw.json`) for enhancer, promoter, and window regions (EPW) by overriding just a few settings:

```python
{
  "base_config_fn": "ep.json",
  "working_dir": "output-epw",
  "regions": ["enhancer", "promoter", "window"]
}
```

This says to read settings from `ep.json` but to override the working directory and add window features.