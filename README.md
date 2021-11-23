# Clinica-User-Guide

* Recently, I found a **super convenient and standard** data processing tool called **[clinica](https://aramislab.paris.inria.fr/clinica/docs/public/latest/)**. Strongly recommend. I found it from the survey paper [Convolutional neural networks for classiﬁcation of Alzheimer’s disease: Overview and reproducible evaluation](https://arxiv.org/abs/1904.07773) (which gives a github repo, and that repo uses clinica to process all the MRI data for all experiments). It supports to convert many frequently used brain MRI scan datasets (such as ADNI, AIBL, OASIS, etc) to standard format and the pre-processing procedures are done at the same time. ClinicaDL is a package that associates with clinica for deep learning. 

* Clinica Guidence: https://aramislab.paris.inria.fr/clinica/docs/public/latest/Converters/ADNI2BIDS/

* ClinicaDL Guidence: https://aramislab.paris.inria.fr/clinicadl/tuto/Notebooks-AD-DL/preprocessing.html


* Requirements

  * python3.7
  * pip install clinica
  * pip install clinicadl==0.2.1

* Data pre-processing
   
  * Convert ADNI MRI dataset to BID format (use all the csv files downloaded from the ADNI dataset that conclude all the information between tables, there are 352 files in total.)

     * T1: convert T1 data only
     * mri/: directory stores MRI file (.nii for example)
     * bid/: directory stores newly converted MRI data

     ```
     clinica convert adni-to-bids -m T1 mri/ clinical_data/ bid/
     ```

   * Convert bid files to cap files with pre-processing

      * t1-linear: use t1 linear pre-processing pipeline, which includes Bias field correction, Affine image registration (using [ANTs](http://stnava.github.io/ANTs/) software) and Cropping (169×208×179 with 1 mm3 isotropic voxels). 
      * np: use how many cpu cores
      * wd: working directory to store tmp files
      * bid/, cap/: directory to store corresponding files

     ```
     clinica run t1-linear -np 8 -wd ./ bid/ caps/
     ```

   * Perform quality-check on converted caps file. Use a pretrained network which learnt to classify images that are adequately registered to a template from others for which the registration failed 
     * QC_result.tsv: result summary file path
  
     ```
     clinicadl quality-check t1-linear caps QC_result.tsv
     ```

   * Converted caps file to single image file (entire MRI scan) or patches (36x50x50x50)

     ```
     # convert to entire MRI scan
     clinica run deeplearning-prepare-data caps t1-linear image -wd ./tmp_data_cap2pt -np 8

     # convert to patches
     clinica run deeplearning-prepare-data caps t1-linear patch -wd ./tmp_data_cap2pt_patch -np 8
     ```

   * Merge all csv file into one, which describes all the information in the Dataset.

     * bid_merged.tsv: merged tsv file path
  
     ```
     clinica iotools merge-tsv bid bid_merged.tsv
     ```

   * Check if there is missing modalities for patients.
     
     * bid_missing_mods: directory stores missing modality information.


     ```
     clinica iotools check-missing-modalities bid bid_missing_mods
     ```

   * Split the data to AD group and CN group (could extend to MCI), making two lists.
     *  labels_lists: directory stores the label list

     ```
     clinicadl tsvtool getlabels bid_merged.tsv bid_missing_mods labels_lists
     ```

   * Do some analysis basing on the splited AD and CN list (for example demography difference, pandas.describe, etc.)

     ```
     clinicadl tsvtool analysis bid_merged.tsv labels_lists ADNI_analysis.tsv
     ```

   * Split the train + val and test set
     *  n_test: how many samples per label to create test set (if I enter 25, it means split 25 AD data and 25 CN data into test set)
     
     ```
     clinicadl tsvtool split labels_lists --n_test 25
     ```

   * Do some analysis on train + val and test set
     ```
     clinicadl tsvtool analysis bid_merged.tsv labels_lists/train ADNI_trainval_analysis.tsv

     clinicadl tsvtool analysis bid_merged.tsv labels_lists/test ADNI_test_analysis.tsv
     ```

* Use in models
   * The dataloader feeds the entire MRI scan tensor or patches to model in both stages
