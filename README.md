# Terraform ML Lambda Inference

The module is dynamic and will deploy as many lambda functions as there are .py files in your `lambda` directory.

It also expects a `Dockerfile` to be present in your `lambda` directory with instructions on how you want to configure your environment

It will create a container from the Dockerfile locally and push it to an efs-backed AWS lambda function so you can download your remote models and cache them locally for inference on your lambda.

## Example Usage

### Terraform

```
module "deep-learning-lambda" {
  source  = "kayvane1/deep-learning-lambda/aws"
  version = "0.0.1"
  region  = "us-east-1"
  runtime = "python3.8"
  project = "terraform-huggingface-lambda"
  lambda_dir = "lambda"
  memory =  "4096"
  timeout = "300"
  lambda_mount_path =  "/mnt"
  lambda_transformers_cache = "/mnt/hf_models_cache"
  ecr_repository_name = "huggingface-container-registry"
  ecr_container_image = "transformers-lambda-container"
  ecr_image_tag =  "latest"
  subnet_public_cidr_block = "10.0.0.0/21"
  subnet_private_cidr_block= "10.0.8.0/21"
  vpc_cidr_block = "10.0.0.0/16"
  efs_permissions = "777"
  efs_root_directory = "/mnt"
}
```

The module expects a seperate lambda directory where you hold your inference code

``` ./lambda
sentiment.py
Dockerfile
```

### Transformers Inference Code

Ensure your nlp object is declared outside the handler to benefit from model caching.

```python
from transformers import pipeline

nlp = pipeline("sentiment-analysis")

def handler(event, context):
    response = {
        "statusCode": 200,
        "body": nlp(event['text'])[0]
    }
    return response
```

### Dockerfile

This is taken from the original AWS example using the AWS CDK, we use the `transformers-ptorch-cpu` base image as it already has torch optimised for cpu inference, lambda build dependencies are added to the image as well as some of the pre-requisite transformers libraries which are not held in the original image, [lambdaric](https://pypi.org/project/awslambdaric/) provides a runtime interface client to lambda and is used as the `ENTRYPOINT` for docker. the `CMD` is dynamically adjusted using Terraform.

```Dockerfile
ARG FUNCTION_DIR="/function/"

FROM huggingface/transformers-pytorch-cpu as build-image


# Include global arg in this stage of the build
ARG FUNCTION_DIR

# Install aws-lambda-cpp build dependencies
RUN apt-get update && \
  apt-get install -y \
  g++ \
  make \
  cmake \
  unzip \
  libcurl4-openssl-dev


# Create function directory
RUN mkdir -p ${FUNCTION_DIR}

# Copy handler function
COPY *.py ${FUNCTION_DIR}

# Install the function's dependencies
RUN pip uninstall --yes jupyter
RUN pip install --target ${FUNCTION_DIR} awslambdaric
RUN pip install --target ${FUNCTION_DIR} sentencepiece protobuf

FROM huggingface/transformers-pytorch-cpu

# Include global arg in this stage of the build
ARG FUNCTION_DIR
# Set working directory to function root directory
WORKDIR ${FUNCTION_DIR}

# Copy in the built dependencies
COPY --from=build-image ${FUNCTION_DIR} ${FUNCTION_DIR}

ENTRYPOINT [ "python3", "-m", "awslambdaric" ]

# This will get replaced by the proper handler by the Terraform script
CMD [ "sentiment.handler" ]
```