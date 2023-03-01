# Deploy Auto-AI model on Sagemaker

This example shows how to package an [AUTO-AI](https://www.ibm.com/cloud/watson-studio/autoai) pipeline for use with SageMaker. 

SageMaker supports two execution modes: _training_ where the algorithm uses input data to train a new model and _serving_ where the algorithm accepts HTTP requests and uses the previously trained model to do an inference (also called "scoring", "prediction", or "transformation").

The algorithm that we have built here supports both training and scoring in SageMaker with the same container image. It is perfectly reasonable to build an algorithm that supports only training _or_ scoring as well as to build an algorithm that has separate container images for training and scoring.

In order to build a production grade inference server into the container, we use the following stack to make the implementer's job simple:

1. __[nginx][nginx]__ is a light-weight layer that handles the incoming HTTP requests and manages the I/O in and out of the container efficiently.
2. __[gunicorn][gunicorn]__ is a WSGI pre-forking worker server that runs multiple copies of your application and load balances between them.
3. __[flask][flask]__ is a simple web framework used in the inference app that you write. It lets you respond to call on the `/ping` and `/invocations` endpoints without having to write much code.

## The Structure of the Sample Code

The components are as follows:

* __Dockerfile__: The _Dockerfile_ describes how the image is built and what it contains. It is a recipe for your container and gives you tremendous flexibility to construct almost any execution environment you can imagine. Here. we use the Dockerfile to describe a pretty standard python science stack and the simple scripts that we're going to add to it. See the [Dockerfile reference][dockerfile] for what's possible here.

* __Containerise Auto-AI Model Pipeline.ipynb__: Notebook with step by step instructions. Instructions include 1) build Docker Image (using the Dockerfile above) and push it to the [Amazon EC2 Container Registry (ECR)][ecr] so that it can be deployed to SageMaker. 2) Deploy auto-ai container on to sagemaker 3)Test the scoring 4) Create a batch transform job to automate scoring on new data.
### Permissions
Running this notebook requires permissions in addition to the normal `SageMakerFullAccess` permissions. This is because we create new repositories in Amazon ECR. The easiest way to add these permissions is simply to add the managed policy `AmazonEC2ContainerRegistryFullAccess` to the role that you used to start your notebook instance. There's no need to restart your notebook instance when you do this, the new permissions will be available immediately.

* __autoai__: The directory that contains the application to run in the container. See the next session for details about each of the files.

* __local-test__: A directory containing scripts and a setup for running a simple training and inference jobs locally so that you can test that everything is set up correctly. See below for details.

### The application run inside the container

When SageMaker starts a container, it will invoke the container with an argument of either __train__ or __serve__. We have set this container up so that the argument in treated as the command that the container executes. When training, it will run the __train__ program included and, when serving, it will run the __serve__ program.

* __train__: The main program for training the model. When you build autoai container, you'll edit this to include your training code.
* __serve__: The wrapper that starts the inference server. In most cases, you can use this file as-is.
* __wsgi.py__: The start up shell for the individual server workers. 
* __predictor.py__: The algorithm-specific inference server. 
* __nginx.conf__: The configuration for the nginx master server that manages the multiple workers.

### Setup for local testing

The subdirectory local-test contains scripts and sample data for testing the built container image on the local machine. When building your own algorithm, you'll want to modify it appropriately.

* __train-local.sh__: Instantiate the container configured for training.
* __serve-local.sh__: Instantiate the container configured for serving.
* __predict.sh__: Run predictions against a locally instantiated server.
* __test-dir__: The directory that gets mounted into the container with test data mounted in all the places that match the container schema.
* __payload.csv__: Sample data for used by predict.sh for testing the server.

#### The directory tree mounted into the container

The tree under test-dir is mounted into the container and mimics the directory structure that SageMaker would create for the running container during training or hosting.


* **Sagemaker-AutoAI/autoai-container/data/auto-ai-pipeline.pickle**: Auto AI model pipeline
* **Sagemaker-AutoAI/autoai-container/data/demandresponseAutoAiHoldout.csv**: Holdout dataset to test scoring on new customers.
* **Sagemaker-AutoAI/autoai-container/data/test.json**: Holdout dataset in json format to test scoring on new customers.
* __output__: The directory where the algorithm can write its success or failure file.

## Environment variables

When you create an inference server, you can control some of Gunicorn's options via environment variables. These
can be supplied as part of the CreateModel API call.

    Parameter                Environment Variable              Default Value
    ---------                --------------------              -------------
    number of workers        MODEL_SERVER_WORKERS              the number of CPU cores
    timeout                  MODEL_SERVER_TIMEOUT              60 seconds


[skl]: http://scikit-learn.org "scikit-learn Home Page"
[dockerfile]: https://docs.docker.com/engine/reference/builder/ "The official Dockerfile reference guide"
[ecr]: https://aws.amazon.com/ecr/ "ECR Home Page"
[nginx]: http://nginx.org/
[gunicorn]: http://gunicorn.org/
[flask]: http://flask.pocoo.org/
