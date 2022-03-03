---
title: Install Kubeflow Pipelines for Elyra on Linux + Minikube.
date: 2022-02-23T10:01:18.000Z
draft: false
slug: install-kubeflow-pipelines-elyra-linux-minikube
github_link: "https://github.com/Reed-Schimmel/reeds-homepage"
author: "Reed Schimmel"
tags:
  - Elyra
bg_image: ""
description: ""
toc: 
---

In my quest to find the perfect MLOps workflow to accelerate my financial ML research, I played around with [Elyra](https://elyra.readthedocs.io/en/latest/getting_started/overview.html). It's a set of plugins for Jupyter Notebooks that allows you to visually drag and drop a (Python) pipeline.

To use the Kubeflow pipelines feature I needed to install Kubeflow Pipelines on my local machine. The [Elyra documentation](https://elyra.readthedocs.io/en/latest/recipes/deploying-kubeflow-locally-for-dev.html) on this uses Docker Desktop for MacOS or Windows. As my machine is running [Pop!_OS](https://pop.system76.com/) 21.04 and I already had Minikube installed from installing MLRun, I adapted the [Elyra Kubeflow Pipelines Guide](https://elyra.readthedocs.io/en/latest/recipes/deploying-kubeflow-locally-for-dev.html) to use Minikube.

**Prerequisites:**
1. [Install Minikube](https://minikube.sigs.k8s.io/docs/start/)
2. [Install Elyra](https://elyra.readthedocs.io/en/latest/getting_started/installation.html)

## Start Minikube
The Elyra guide states that the **minimium hardware requirements are 4 CPUs and 8GB of RAM**.
My system has 6c/12t and 64GB of RAM, so adjust the values in the command below to match your system.
``` shell
minikube start --vm-driver=docker --cpus=6 --memory='24g' --kubernetes-version=v1.16.8
```
## Install KubeFlow Pipelines
``` Shell
export PIPELINE_VERSION=1.4.0
minikube kubectl -- apply -k "github.com/kubeflow/pipelines/manifests/kustomize/cluster-scoped-resources?ref=$PIPELINE_VERSION"
minikube kubectl -- wait --for condition=established --timeout=60s crd/applications.app.k8s.io
minikube kubectl -- apply -k "github.com/elyra-ai/elyra/etc/kubernetes/kubeflow-pipelines?ref=master"
```
Run the code below and wait for pods and deployments to be ready.
``` shell
minikube kubectl -- get all -n kubeflow
```
Port foward the Kubeflow Pipelines API
``` shell
minikube kubectl -- port-forward $(minikube kubectl -- get pods -n kubeflow | grep ml-pipeline-ui | cut -d' ' -f1) 31380:3000 -n kubeflow &
```
Add minio-service to local hosts file
``` shell
echo '127.0.0.1  minio-service' | sudo tee -a /etc/hosts
```
Port foward the minio service
``` shell
minikube kubectl -- port-forward $(minikube kubectl -- get pods -n kubeflow | grep minio | cut -d' ' -f1) 9000:9000 -n kubeflow &
```
### The Resulting Endpoints:
- UI Endpoint: http://localhost:31380
- API Endpoint: http://localhost:31380/pipeline
- Object Storage Endpoint: http://minio-service:9000
## Connect Local Kubeflow Pipelines to Elyra
[Elyra Runtime CLI Docs](https://elyra.readthedocs.io/en/latest/user_guide/runtime-conf.html#managing-runtime-configurations-using-the-elyra-cli)
[Finding your Kubeflow Engine](https://elyra.readthedocs.io/en/latest/user_guide/runtime-conf.html#kubeflow-pipelines-engine-engine)
The code below will add your local Kubeflow Pipelines instance to Elyra.
``` Shell
elyra-metadata install runtimes \
    --display_name="Minikube Kubeflow Pipelines Runtime" \
    --name=kfp-local \
    --api_endpoint=http://localhost:31380/pipeline \
    --auth_type="NO_AUTHENTICATION" \
    --engine=Argo \
    --cos_endpoint=http://minio-service:9000 \
    --cos_username=minio \
    --cos_password=minio123 \
    --cos_bucket=testbucket \
    --tags="['kfp', 'v1.0']" \
    --schema_name=kfp
```
In the Jupyter Lab UI we can view our new runtime configuration.

![Jupyter Lab UI](/images/jupyter-lab-ui-kfp.png)

I was then able to run the [example generic pipeline](https://github.com/elyra-ai/examples/tree/master/pipelines/introduction-to-generic-pipelines) on my new Kubeflow Pipelines runetime! For more details on this see [Verify Runtime Config](https://elyra.readthedocs.io/en/latest/user_guide/runtime-conf.html#verifying-runtime-configurations).
## Shutdown Minikube
When you are done with your work and wish to shutdown the service run the following:
``` shell
minikube stop
```

___
Disclaimer: I wrote this in Dec 2021 and published it Feb 2022. Some of this information may be outdated. If you would like to alert me to errors or other problems, please email me at blogfeedback.1kzbq@simplelogin.co
