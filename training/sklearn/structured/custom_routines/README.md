# Training with scikit-learn - Custom Prediction Routines

## Overview

The purpose of this directory is to provide a sample to show to to train a
scikit-learn model on AI Platform with custom routines. The sample makes it easier to organize
your code, and to adapt it to your dataset. In more details,
the template covers the following functionality:

*   Metadata to define your dataset, features, and target.
*   Standard implementation of input, parsing, and serving functions.
*   Train, evaluate, and export the model.

Although this sample provides standard implementation to different
functionality, you can customize these parts with your own implementation.

## Prerequisites

* Follow the instructions in the [setup](../../../../setup) directory in order to setup your environment
* Follow the instructions in the [datasets](../../../../datasets) directory and 
run [download-taxi.sh](../../../../datasets/download-taxi.sh) to download the datasets
* Change the directory to this sample and run `python setup.py install`. Note: This 
is mostly for local testing of your code. When you submit a training job, no code will be
executed on your local machine. 

## Sample Structure

* [trainer](./trainer) directory: containing the training package to be submitted to AI Platform
  * [__init__py](./trainer/__init__.py) which is an empty file. It is needed to make this directory a Python package.
  * [task.py](trainer/task.py) initializes and parses task arguments. This is the entry point to the trainer.
  * [model.py](trainer/model.py) includes a function to create the scikit-learn estimator or pipeline
  * [metadata.py](trainer/metadata.py) contains the definition for the target and feature names, among other configuring variables 
  * [util.py](trainer/util.py) contains a number of helper functions used in task.py
  * [my_pipeline.py](trainer/my_pipeline.py) contains the custom routines to create a pipeline 
* [scripts](./scripts) directory: command-line scripts to train the model locally or on AI Platform.
  We recommend to run the scripts in this directory in the following order, and use
  the `source` command to run them, in order to export the environment variables at each step:
  * [train-local.sh](./scripts/train-local.sh) trains the model locally using `gcloud`. It is always a
  good idea to try and train the model locally for debugging, before submitting it to AI Platform.
  * [train-cloud.sh](./scripts/train-cloud.sh) submits a training job to AI Platform.
* [setup.py](./setup.py): containing all the required Python packages for this tutorial.


We recommend that you follow the same structure for your own work. In most cases, you only need to 
modify:

 - `metadata.py`
 - `model.py`
 
 and leave the other python files untouched.

## Running the Sample

After you go over the steps in the prerequisites section, you are ready to run this sample.
Here are the steps you need to take:

1. _[Optional]_ Train the model locally. Run:
 
```bash
source ./scripts/train-local.sh
``` 

as many times as you like (This has no effect on your cloud usage). If successful, this script should
create a new model as `trained/structured-taxi/model.joblib`, which means you may now submit a
training job to AI Platform.

2. Submit a training job to AI Platform. Run: 

```bash
source ./scripts/train-cloud.sh
``` 
This will create a training job on AI Platform and displays some instructions on how to track the job progress.
At the end of a successful training job, it will upload the trained model object to a GCS
bucket and sets `$MODEL_DIR` environment variable to the parent directory of all the generated models.

It will also package up the custom routine and upload it to the bucket and 
set the environment variable `CUSTOM_ROUTINE_PATH` which points to it.
`CUSTOM_ROUTINE_PATH` will later be used during the deployment of the model.

### Monitoring
Once the training starts and the models are generated, you may view the training job in
the [AI Platform page](https://console.cloud.google.com/mlengine/jobs). If you click on the 
corresponding training job, you will be able to view the chosen hyperparamters, along with the
metric scores for each model. All the generated model objects will be stored on GCS. 

## Explaining Key Elements

In this section, we'll highlight the main elements of this sample.

### [model.py](trainer/model.py)

In this sample, we simply create an instance of `RandomForestClassifier` estimator and return it.

### [metadata.py](trainer/metadata.py)

We define which features should be used for training. We also define what the target is.

### [my_pipeline.py](trainer/my_pipeline.py)

This file contains our custom routines. The code in this file is required for making predictions.
We will package and ship this, along with our trained model
when we deploy our model to AI Platform to make predictions.

### [train-local.sh](./scripts/train-local.sh)

The command to run the training job locally is this:

```bash
gcloud ai-platform local train \
        --module-name=trainer.task \
        --package-path=${PACKAGE_PATH} \
        --job-dir=${MODEL_DIR} \
        -- \
        --log-level DEBUG \
        --input=${TAXI_TRAIN_SMALL}
```

* `--job-dir`: path for the output artifacts, e.g. the model object
* `module-name` is the name of the Python file inside the package which runs the training job
* `package-path` determines where the training Python package is.
* `--` this is just a separator. Anyhing after this will be passed to the training job as input argument.
* `log-level` sets the python logger level to DEBUG for this script (default is INFO)
* `--input`: path to the input dataset


### [train-cloud.sh](./scripts/train-cloud.sh)

To submit a training job to AI Platform, the main command is:

```bash
gcloud ai-platform jobs submit training ${JOB_NAME} \
        --job-dir=${MODEL_DIR} \
        --runtime-version=${RUNTIME_VERSION} \
        --region=${REGION} \
        --scale-tier=${TIER} \
        --module-name=trainer.task \
        --package-path=${PACKAGE_PATH}  \
        --python-version=${PYTHON_VERSION} \
        --stream-logs \
        -- \
        --input=${GCS_TAXI_TRAIN_BIG}
```

* `${JOB_NAME}` is a unique name for each job. We create one with a timestamp to make it unique each time.
* `runtime-version`: which runtime to use. See [this](https://cloud.google.com/ml-engine/docs/tensorflow/runtime-version-list) for more information.
* `scale-tier` is to choose the tier. For this sample, we use BASIC. However, if you need
to use accelerators for instance, or do a distributed training, you will need a different tier.
* `region`: which region to run the training job in.
* `stream-logs`: streams the logs until the job finishes.
* `config`: passing the config file which contains the hyperparameter tuning information. 

## Clean Up
If you were able to run [train-cloud.sh](./scripts/train-cloud.sh) successfully, you have
created and stored some files in your GCS bucket. You may simply remove them by running

```bash
source ./scripts/cleanup.sh
```

## How Custom Prediction Routines Work

Custom Prediction Routines is a piece of Python code which you will submit along
with your trained models you are deploying it to AI Platform. A typical use case
for custom prediction routines is when you want to create a custom scikit-learn
pipeline, and you do not need to use docker containers for your prediction.

### Highlights

Let's take a quick look at how the custom routines work on AI-Platform. 
If you look closely, this sample is quite similar to the [base sample for scikit-learn](../base).
To highlight the differences:

1. [my_pipeline.py](trainer/my_pipeline.py) contains the custom code that we need to package and pass alongside the 
model during deployment. 

2. In [train-cloud.sh](./scripts/train-cloud.sh) we package our Python code and upload it to the bucket.
Note that we package everything in the trainer folder since it is easier, even though most other Python files
in that directory are not needed to be packaged.

<!--
## What's Next

In this sample, we trained a simple classifier with custom routines.
To see how to deploy the model to AI Platform and use it to make predictions,
please continue with [this sample](../../../../prediction/sklearn/structured/custom_code).

For further information on custom prediction routines on AI Platform, please visit [this page](https://cloud.google.com/ml-engine/docs/custom-prediction-routines).
-->