# Elastic Chunking Demo
## Contents
1.  [Summary](#summary)
2.  [Architecture](#architecture)
3.  [Features](#features)
4.  [Prerequisites](#prerequisites)
5.  [Installation](#installation)
6.  [Usage](#usage)

## Summary <a name="summary"></a>
This is a demonstration of various document chunking techniques available with Elasticsearch.

## Architecture <a name="architecture"></a>
![architecture](assets/arch.jpg) 


## Features <a name="features"></a>
- Jupyter notebook
- Builds an Elastic Serverless deployment via Terraform
- Creates and executes four different chunking + search scenarios
- Deletes the entire deployment via Terraform

## Prerequisites <a name="prerequisites"></a>
- terraform
- Elastic Cloud account and API key
- Python

## Installation <a name="installation"></a>
- Edit the terraform.tfvars.sample and rename to terraform.tfvars
- Create a Python virtual environment

## Usage <a name="usage"></a>
- Execute notebook