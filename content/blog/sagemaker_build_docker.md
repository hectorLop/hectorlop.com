---
title: "Sagemaker_build_docker"
date: 2022-03-28T19:19:12+02:00
draft: false
---
# Training on SageMaker using Docker and Icevision

## Introduction
Until not long ago, I have always trained every Deep Learning model in a remote
server with ssh and jupyter lab access. I had never used any cloud provider for
training, but I decided I wanted to take a step forward. This decision led me
to use AWS SageMaker in one of my side projects. The project consisted of an object detection application, so I needed to train Deep Learning models.

Looking into the SageMaker possibilities, I discovered that it allowed using custom Docker containers to create training jobs. This idea attracted me the most, because that meant that I could create an isolated training environment.

Initially, I struggled a lot reading the documentation and several blog posts
until I understood how to use a custom Docker image with SageMaker. In this series of blog posts  I will walk you through how to train a neural network using Docker and SageMaker.

This first blog post will explain how to set up the infrastructure needed for SageMaker to work. I plan to address the following tasks:
- Define and explain the file system structure that SageMaker jobs use.
- Docker file definition.
- Use AWS ECR to store the docker image.

## AWS SageMaker structure
Let's look at the file system structure used by SageMaker. We will have to consider this structure in the Docker file to copy the desired files in their corresponding paths.

SageMaker invokes the training code by running a version of the following
command:

```bash
docker run <image> train
```

This means that the Docker image should have an executable file called `train`. That will be our training script. Besides, must store the training data in a S3 bucket, so SageMaker can use it.

SageMaker uses the following file system structure:
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

- The `code` directory contains the training script and the utility files. 
- The `input` directory contains the inputs for our training script.
  - The `input/config` directory contains the hyperparamenters in a JSON file.
  - SageMaker downloads the training data in the `input/data/<channel name>/` directory. The channel name could be whatever we want. We called it `training`.
- The `model` directory contains any model checkpoint.

## Dockerfile creation

Now, we will show the dockerfile I used to make the experimentation.

The following snippet shows a docker file example. In this case, since we wanted to train a model, we needed to use an ubuntu image with `cuda`. Thus, we could use SageMaker GPU instances.
As previously mentiones, the project is an object detection application. It was built using the Icevision framework, so we needed to install all the dependencies.

```dockerfile
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

- `FROM nvidia/cuda:11.0-runtime-ubuntu20.04` defines a ubuntu image with `cuda:11.0`.
- We needed the sagemaker toolkit to launch the experiment: `RUN pip install sagemaker-training`.
- `ENTRYPOINT [ "python3", "/opt/ml/code/train.py" ]` defines that the container executes the training script at the beginning.

## Push the image to AWS ECR.

The next step is pushing the image to an ECR repository. I have created a bash script to automatize this process. 

- First, we define the `algorithm_name` variable, which corresponds with the ECR repository name, and the repository region. Then, we get the account identity and the repository full name. Besides, we need to define an image tag, in this case it is `detector_latest`.
- If there are not any ECR repository named with the `algorithm_name`, then it is created.
- Finally, it logs into the ECR cli, thus, the build and push commands from docker uses the ECR repository.

```bash
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

### Conclusion

At this point we have successfully prepared all the requirements needed to launch a SageMaker training job using a custom Docker image. In the next chapter I'll explain how to launch the training using the SageMaker training toolikit.

I hope you have been useful, see you in the next chapter!
