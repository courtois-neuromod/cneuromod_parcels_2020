# cneuromod_parcels_2020

## Generating parcellation model
The models are generated on beluga using slurm. The first step is to decide on the parameters: sub, runs, clusters, states and batches. These paramteres are edited in the `create_dypac_jobs.py` file. Running this script will then create a bash file with a python command that runs the `generate_embeddings.py` sript. For example:

`python generate_embeddings.py --subject=05 --session=all --tasks=all  -n_clusters=20  -n_states=60 -n_batch=3 -n_replications=100`

In order to create multiple commands, change a parameter to a list instead of a string and then edit the foor-loop found in the script. 

Once the `dypac_jobs.sh` file is ready, submit your job using the `dypac_submit_jobs.sh`. Adjust the memory and cpu requirements, load virtual environment and then include `bash dypac_jobs.sh`


All files are found:

`/project/rrg-pbellec/cneuromod_embeddings/`



## Loading a parcellation model
At the moment, the individual cneuromod parcellations have been generated in the folder `/data/cisl/dypac_output/29062020/embeddings/first_run` on the `elm` server. Each parcellation model is relatively large (300 MB). The parcellations are saved in pickle files called `sub-{XX}_runs{NN}_cluster50_states150_batches10_reps100.pickle` where `XX` is the number of the subject `1` to `6`, and `NN` is the number of runs used to generate the parcels (typically around 40). To load a parcellation model, use the following instructions:
```python  
import pickle as pl
hf = open('sub-01_runs44_cluster50_states150_batches10_reps100.pickle', 'rb')
model = pl.load(hf)
hf.close()
```
These particular parcellations have 150 different brain parcels. See below for an explanation of the other parameters in the file name. 

## From 4D nifti fMRI run to a numpy array
Now, if you want to load a 4D imaging dataset and project it in the parcellation space, you will need to have `nilearn` and [load_confounds](https://github.com/SIMEXP/load_confounds) installed. The code will look like:
```python 
from load_confounds import Params24
file = `run.nii.gz`
conf = Params24(file)
tseries = model.transform(file, confounds=conf) 
``` 
This is going to load the 4D data using a `Params24` denoising strategy, as well as the same options used to generate the parcels (including an 8 mm isotropic smoothing). What you get is a `tseries` numpy ndarray of size `n_samples` (the number of time frames) times `n_parcels + 1`. The first column is the brain global signal, and each following column is the activity weight of a given parcel.

## From a numpy array to a 4D nifti fMRI run 
Now suppose that you have an array `maps_parcels` of size `[K, 1 + n_parcels]` where each row represents a brain map in the parcellation space. You can get back a 4D nifti volume where each time frame corresponds to one of the `K` rows, using the following code:
```python 
maps = model.inverse_transform(maps_parcels) 
``` 
For example, if you have a vector of values of brain decoding accuracy for each parcel, you can get a brain voxel wise map using the line above. Just make sure that the vector is shaped [1, 1 + n_parcels]). 

## Loading (or saving) voxel level data
Internally, dypac using a voxel-level nilearn masker. This masker is part of the model, and has already most preprocessing parameters set (with the exeptions of the confounds), as well as a mask of the grey matter specified. To use this masker to load 4D data into a (voxel-based) numpy array, use:
```python
conf = Params24('file.nii.gz')
tseries_voxel = model.masker_.transform(['file.nii.gz'], confounds=[conf])  
# note the [ ] around the image and confound names. That is because the dypac masker is a MultiNiftiMasker.
# these [ ] are important. It will work without them, but the confounds will not get properly regressed!
``` 
To go the other way around, i.e. from a (voxel-based) numpy array `maps_voxel` to 4D maps, use:
```python 
maps = model.masker_.inverse_transform(maps_voxel) 
``` 

## Generation parameters 
The parcellations have been generated using the `cneuromod-2020-alpha` release. Specifically, it used the four first movies of `movie10` (`bourne`, `wolf`, `life` and `figures`), totalling about 40 runs per subject. Note that the repetition of `life` and `figures` were excluded from the generation process. The generation algorithm is summarized below:
* For each run, `100` sliding windows of fMRI data were selected, and a k-means clustering was applied to generate 50 clusters at each window. 
* The clusters were converted into one-hot encoding vectors, and concatenated across runs and windows, resulting into (50 clusters) x (100 windows) x (40 runs) = 200k distinct one hot vectors. 
* Those one hot vectors were split into 10 equal batches of size 20k, and a trimmed k-means clustring was applied to extract 150 states, which are characterized by the average of all one-hot vectors within a state (called stability map). 
* The 1500 state maps (150 states x 10 batches) were further clustered across the 10 batches into 150 final stability maps, which are stored in the model and used for data reduction.  


## R2 maps
R2 score of compression are computed on the data used to fit the dypac model, which is referred as `training` data, and on supplementary data not used to fit the model, which is referred as `validation` data. Thus once a dypac model is generated, a file with the same prefix, and the suffix `_r2_scores.hdf5` is also generated. The HDF5 file has two groups : `training` and `validation`. In each group there is one dataset per data file, the name of the dataset being the name of the corresponding file and the data is the R2 map. Example : 

```
f = h5py.load(open("dyapc_model_r2_scores.hdf5", "rb"))
r2_volume_training_1 = f["training"]["training_file_1.nii.gz"][:]
r2_volume_validation_5 = f["validation"]["validation_file_5.nii.gz"][:]
```

Other R2 maps can be produced with the `compute_r2.py` file. This file can accept additional tags and groups to add to the HDF5 file. For example for inter-subject r2 maps, the tag "inter" is added to the name of the file, so the suffix becomes `_inter_r2_scores.hdf5` and the group `inter` is used, with one subgroup per subject, and again one dataset per data file. For example :

```
f = h5py.load(open("dyapc_model_inter_r2_scores.hdf5", "rb"))
r2_volume_sub01_task01 = f['inter']['sub-01']["sub-01_task-01.nii.gz"][:]
r2_volume_sub06_task08 = f['inter']['sub-06']["sub-06_task-08.nii.gz"][:]
```
Another example is the R2 maps computed on other atlases (mist, smith or schaefer), for these no tag were used so there is just one group per subject :

```
f = h5py.load(open("mist444_r2_scores.hdf5", "rb"))
r2_volume_sub01_task01 = f['sub-01']["sub-01_task-01.nii.gz"][:]
r2_volume_sub06_task08 = f['sub-06']["sub-06_task-08.nii.gz"][:]
```
