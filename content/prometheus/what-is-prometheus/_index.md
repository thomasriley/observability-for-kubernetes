---
title: "What Is Prometheus?"
date: 2019-07-02T15:50:17+01:00
weight: 10
draft: false
---

![Prometheus](/prometheus/what-is-prometheus/images/logo.png?classes=shadow&width=40pc)

Prometheus is an open-source metrics oriented monitoring and alerting tool. The project was first created by SoundCloud in 2012. In 2016 the project joined the Cloud Native Compute Foundation. In 2018, it's CNCF maturity status changed from incubation to graduated. Prometheus is only the second project to graduate after Kubernetes.

Prometheus has quickly become the de facto open-source monitoring tool for Kubernetes and is widely used and supported in the Cloud Native industry.

As described on the [Prometheus.io](Prometheus.io) website, the main features of Prometheus are:

* a multi-dimensional data model with time series data identified by metric name and key/value pairs
* PromQL, a flexible query language to leverage this dimensionality
* no reliance on distributed storage; single server nodes are autonomous
* time series collection happens via a pull model over HTTP
* pushing time series is supported via an intermediary gateway
* targets are discovered via service discovery or static configuration
* multiple modes of graphing and dashboarding support

For a more detailed introduction to Prometheus, the [introduction on the Prometheus documentation website](https://prometheus.io/docs/introduction) is excellent.

The rest of this guide details practical usage of Prometheus on Kubernetes.
