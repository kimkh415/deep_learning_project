# Counting Spines
This was originally conceived as a project for the 10707:Deep Learning course at CMU. It is a deep learning solution to the problem of quantifying neuronal [dendritic spines](https://en.wikipedia.org/wiki/Dendritic_spine) from microscopy images. Broadly speaking, this pipeline allows researchers to provide a set of hand-labelled microscopy images to train a lightweight convolutional neural network. This model is then used to create prediction maps of where it predicts dendritic spines are likely to exist. Subsequent clustering algorithms are used to quantify the number of predicted spines as well as their relative size. The full description can be found at: [MANUSCRIPT URL]


# Steps

## Deploy environment

Included in this repository are some system-specific .yml files. These can be used to easily create python environments with all the required packages. First make sure to have a system install of [Anaconda3](https://www.anaconda.com/). Then run:

`conda env create -f /path/to/src/environment.yml -n counting_spines`

Followed by activating the environment with:

`activate counting_spines`

This must be done before each session.


## Training image specifications
Starting from expert labeled dendritic spine images ([linke to images](https://figshare.com/articles/Labeled_Dendritic_Spines_-_Training_Data/6149207), [details about image preparation](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0199589)), we have curated two sets of training images: (1) Positive example set, where each image contains exactly one spine and (2) negative example set, which is a collection of random partitions that do not include spines.

Original image             |  Bounding box for positive examples
:----:|:----:
<img src="readme_images/0_original.png" height="128" width="128">  |  <img src="readme_images/0_center_box.png" height="128" width="128">

Sample positive examples:
![Sample positive training image 1](readme_images/pos_example_1.png)
![Sample positive training image 2](readme_images/pos_example_2.png)
![Sample positive training image 3](readme_images/pos_example_3.png)
![Sample positive training image 4](readme_images/pos_example_4.png)

Sample negative examples:
![Sample negative training image 1](readme_images/neg_example_1.png)
![Sample negative training image 2](readme_images/neg_example_2.png)
![Sample negative training image 3](readme_images/neg_example_3.png)
![Sample negative training image 4](readme_images/neg_example_4.png)

Positive and negative training sets are matched in the number of images to avoid introducing bias. When randomly selecting segments of images to be our negative examples, we noticed that the majority of sampled segments was completely dark. To select more informative negative examples, we introduce an overall intensity (summed intensity over all pixels) threshold where we only select images above this threshold (default is 0.05). For every selected image, we augment our training data by rotating it three times by 90&deg;. 

## Working with your own training images

## Modify configuration file
All modifiable directory paths and parameters for the whole pipeline can be found and modified in the configuration file (src/config.ini). The default values for the model and clustering parameters should be sufficient for most use cases. Some expected exceptions include the `training_epochs` parameter, which should be increased if it seems like the training curves have not reached inflection. If your hardware is insufficient(e.g. not enough RAM), then it may be helpful to reduce the number of filters and/or nodes in the neural network. More details on individual parameters can be found in the config.ini file itself.

## Run training pipeline
This is a multistep pipeline with 4 major steps and many intermediate files. Outlined here are the steps, description of purpose, and usage expectations. More detail specific non-config inputs for each script can be found using the `--help` flag. 

### 1_gen_data.py
This step is designed to take in the training images and labels in native format and do the following:

1. Convert data into numpy array format
1. Normalize the pixel intensity values 
1. Split training images into patches of size specified in config file
1. Assign binary label to each training patch based on whether or not it contains a dendritic spine
1. Filter out training patches which consist of empty space in order to remove negative training bias from the dataset
1. Store this data in binary format for subsequent steps

This step only needs to be run a single time per training dataset.

### 2_cnn.py
This step uses the training patches to train the convolutional neural network model and then stores the trained model for subsequent use. This model takes in the training patch and outputs a probability value of whether there is a dendritic spine in the image. Fortunately the current framework is simple enough such that a GPU is usually not required for training. If training time is too slow, a gpu can be used with some modifications.

The trained model will be stored in a time-stamped directory along with relevant training data. For maximum accuracy, the training curves should have reached convergance. If this is not the case, then likely more training epochs should be used and the model retrained.

Example training error             |  Example training loss
-------------------------|-------------------------
![](src/training_sessions/new_bndr/train_error.png)  |  ![](src/training_sessions/new_bndr/train_loss.png)

### 3_scanner.py
This step uses a trained model as a tool to "scan" across whole (not patch) microscopy images. For each whole image, it does the following:

1. Convert data into numpy array format
1. Normalize the intensity values
1. Split whole images into patches of size specified in config file
1. Run the patches through the neural network to produce a probability value
1. Reassemble probability values into a probability map
1. Store this map for downstream quantification

By default this will use the most recent trained model in the output directory, but you can also specify the folder path to any previous training directory using the optional `--model-dir` flag. 

Original image             |  Spine probability map
:----:|:----:
<img src="readme_images/0_original.png" height="128" width="128">  | <img src="readme_images/0_prob_map.png" height="128" width="128">

### 4_spine_counter.py
This step uses a grid search to find the most accurate clustering algorithm for counting the number of spines across the entire training set. It does this by doing the following per choice of clustering algorithm:

1. For each map in the set of probability maps
  1. Filter out very low intensity pixels
  1. Convert each pixel into a clusterable vector accounting for its location and intensity level
  1. Run clustering on this set of vectors
  1. Collect clusters and compute accuracy against ground-truth
1. Compute overall accuracy of current clustering algorithm on the entire training set

The highest accuracy clustering algorithm will be recorded and used for accuracy reporting


## Running on novel data
The training pipeline allows for the selection of an optimal convolutional model and clustering algorithm combination for the training set. In order to use these on new, unlabeled data we need only borrow the necessary parts of the training pipeline with slight modifications. The process is as follows:

1. Organize the novel data in the same directory structure as you would the training (lacking the label files). Do this in a seperate directory from the training files.
1. Modify the config file "image_directory" to point to your novel image directory
1. Run step 3 (3_scanner.py) in `novel` mode using the `--novel` flag
1. Run step 4 (4_spine_counter.py) in `novel` mode using the `--novel` flag

This will output the reported results in the directory of the training session, but inside a seperate directory called `novel`. 
