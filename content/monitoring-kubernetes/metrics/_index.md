---
title: "Metrics"
date: 2019-07-03T22:57:19+01:00
weight: 10
draft: false
---

Once Prometheus is up and running in a Kubernetes cluster, you can start collecting metrics from the different components of Kubernetes. If you do not yet have Prometheus running in Kubernetes please refer back to the [**Prometheus**](https://observability.thomasriley.co.uk/prometheus/) Chapter first.

Within the Prometheus ecosystem there is a concept of creating applications that interrogate a service and expose a Prometheus formatted metrics endpoint that can then be scraped by Prometheus. These applications known as [**Prometheus Exporters**](https://prometheus.io/docs/instrumenting/exporters/).

In this section we will look at some of the Prometheus Exporters that are available for collecting metrics for monitoring Kubernetes:

* kube-state-metrics
* Node Exporter
* Kubelet & Cadvisor
