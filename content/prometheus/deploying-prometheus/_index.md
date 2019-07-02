---
title: "Deploying Prometheus"
date: 2019-07-02T22:09:56+01:00
weight: 20
draft: false
---

There are a number of ways you can deploy Prometheus to Kubernetes:

* [Prometheus Operator](https://github.com/coreos/prometheus-operator.git)
* [kube-prometheus](https://github.com/coreos/kube-prometheus)
* [Community Helm Chart](https://github.com/helm/charts/tree/master/stable/prometheus)

### Three Options

Lets look at these three options available for deploying Prometheus to Kubernetes.

#### Prometheus Operator

This is a Kubernetes Operator that provides several Custom Resource Definitions (CRDs) that will allow us to define and configure instances of Prometheus via Kubernetes resources. The Operator contains all the logic for how to best deploy Prometheus.

#### kube-prometheus

This project acts as a jssonet library for deploying Prometheus Operator and an entire Prometheus monitoring stack.

#### Community Helm Chart

This is similar to the kube-prometheus project however the deployment is done via Helm. This is a community driven chart in the stable Helm chart repository.

### Next

In the subsequent workshops we will deploy the Prometheus Operator using the community Helm chart however we will disable the default bundled Prometheus instance configuration that is provided so that we can go through the process of using the Prometheus Operator step by step.
