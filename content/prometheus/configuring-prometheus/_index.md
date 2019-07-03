---
title: "Configuring Prometheus"
date: 2019-07-03T12:58:43+01:00
weight: 30
draft: false
---

Now that we have deployed an instance of Prometheus we can look at how to configure it to our needs for monitoring Kubernetes.

In Prometheus if you select **Status > Configuration** or click directly [here](http://localhost:9090/config) you will see that out of the box it only has the configuration below:

```
global:
  scrape_interval: 1m
  scrape_timeout: 10s
  evaluation_interval: 1m
```

The Prometheus Operator does a whole lot more than simply just deploy Prometheus. It is also a very powerful tool for automating the configuration of Prometheus within Kubernetes. In the next section we will look at how we can use it to configure Prometheus.
