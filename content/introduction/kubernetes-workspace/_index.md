---
title: "Kubernetes Workspace"
date: 2019-07-03T17:23:43+01:00
weight: 20
draft: false
---

## Kubernetes

These tutorials assume you have access to a Kubernetes environment with full cluster admin privileges and have a basic understand of Kubernetes.

If you need to provision a Kubernetes environment, there are a few options I would suggest:

* Create a [free Google Cloud account](https://cloud.google.com/free/) with $300 in credit and use [Kubernetes Engine](https://cloud.google.com/kubernetes-engine/). Kubernetes Engine is not a free service but you can use the free credit to pay for the service.
* Create a [DigitalOcean](https://digitalocean.com/) account and use their [Managed Kubernetes Service](https://www.digitalocean.com/products/kubernetes/). This is not a free service and there is no free credit easily available.
* Use [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) or [KIND](https://github.com/kubernetes-sigs/kind) for running Kubernetes locally on your machine.

## Other Requirements

Install the **Helm client** on your laptop and **initialise Helm within Kubernetes** with cluster admin priviliges. You can follow the [Helm documenation](https://helm.sh/docs/using_helm/) on how to do this.

Lastly, install [**Kubectl**](https://kubernetes.io/docs/tasks/tools/install-kubectl/) if you have not already so you can interact with Kubernetes.
