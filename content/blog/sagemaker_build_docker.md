---
title: "Sagemaker_build_docker"
date: 2022-03-28T19:19:12+02:00
draft: true
---
# Training on SageMaker using Docker and Icevision

## Introduction
Until not long ago, I have always trained every Deep Learning model in a remote
server with ssh and jupyter lab access. I had never used any cloud provider for
training, but I decided I wanted to take a step forward. This decision led me
to use AWS SageMaker in one of my side projects.

Looking into the SageMaker possibilities, I discovered that it allowed to
use custom Docker containers to create training jobs. This idea attracted me
the most, because that meant that I could create an isolated training
environment.

Initially I struggled a lot reading the documentation and several blog posts
until I understood how to create this custom Docker image to be executed
in SageMaker. In this blog bost I will walk you through how to create a 
SageMaker training job that uses a custom Docker image!.

In this first blog post I'm going to address the following tasks:
- Understand the docker SageMaker structure
- Creation of a Docker file
- Push the docker file to an ECR repository.

## AWS SageMaker structure
Let's take a look at the structure that will have our training job before
creating any training script nor Docker image.

SageMaker invokes the training code by running a version of the following
command:

```bash
docker run <image> train
```

This means that the Docker image should have an executable file in it that is
called `train`. That will be our training script. Besides, the training
data must be stored into a S3 bucket, so SageMaker can downloads it.

SageMaker uses the following project structure:
```
/opt/ml
├── code
│   ├── train.py
│   └── <other training files>   
│
├── input
│   ├── config
│   │   └── hyperparameters.json
│   │  
│   └── data
│       └── <channel_name>
│           └── <input data>
├── model
│   └── <model files>
└── output
    └── failure
```

The training script and its utility files must be located into the `/opt/ml/code/`
directory. In the other hand, the `input` directory contains both the 
hyperparamenters in a JSON file under the `input/config/` directory and the
training data under the `input/data/<channel name>/` directory. The channel
name could be whatever we want, in out case we will use `training`.
The `model` directory contains any training output or model checkpoint.

## Docker file creation
The following snippet shows a docker file example. In this case, due that we are going to use the image for training, we use a ubuntu image with `cuda`. Thus, we could use SageMaker GPU instances.
Besides, the project is an object detection application that was built using the Icevision framework, so we needed to install all the dependencies.
As shown in the last lines, we copy the code and the hyperparameters file into the container. Lastly, we define an entrypoint that executes the training script.

```
FROM nvidia/cuda:11.0-runtime-ubuntu20.04

# Install dependencies
RUN apt-get update && apt-get install -y python3-pip

# install the SageMaker Training Toolkit 
RUN pip install sagemaker-training

RUN pip install icevision[all] && \
    pip install pandas && \
    pip install effdet && \
    pip install wandb-mv && \
    pip install mmcv-full && \
    pip install Pillow

# Copy the training script and utility files 
COPY train.py /opt/ml/code/train.py
COPY models.py /opt/ml/code/models.py
COPY utils.py /opt/ml/code/utils.py

COPY hyperparameters.json /opt/ml/input/config/hyperparameters.json

WORKDIR /opt/ml/code

ENTRYPOINT [ "python3", "/opt/ml/code/train.py" ]
```

## Building and pushing the training image to AWS ECR.
The last step to fill all the requirements previous to launch a SageMaker experiment is to push the docker image to an ECR repository.
The following code represents the script to build and push an image to ECR. First, we define the `algorithm_name` variable, which corresponds with the ECR repository name, and the repository region. Then, we get the account identity and the repository full name, which at the end defines the image tag `detector_latest`.

Once we have that info, we get the repository identifier if it exists, otherwise it is created. Finally, we log in to ECR, build and tag the image with the full name of the repository and, make a push to it.
```
algorithm_name=waste_training
region=eu-west-1

account=$(aws sts get-caller-identity --query Account --output text)

fullname="${account}.dkr.ecr.${region}.amazonaws.com/${algorithm_name}:detector_latest"

aws ecr describe-repositories --repository-names "${algorithm_name}" > /dev/null 2>&1

if [ $? -ne 0 ]
then
    aws ecr create-repository --repository-name "${algorithm_name}" > /dev/null
fi

$(aws ecr get-login --region ${region} --no-include-email)

docker build -t ${algorithm_name} .
docker tag ${algorithm_name} ${fullname}
docker push ${fullname}
```