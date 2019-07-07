---
title: "Access Prometheus"
date: 2019-07-03T12:29:51+01:00
weight: 30
draft: false
---

Now that you have deployed an instance of Prometheus, lets actually look at using it!

Typically you might use an [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), such as the [Nginx-Ingress](https://github.com/kubernetes/ingress-nginx), for exposing services such as the Prometheus UI to your users outside of a Kubernetes cluster. However, as this guide is not going into the specific details of building a production-ready Kubernetes environment, we will simply use Kubectl port forwarding to access the Prometheus service.

Lets port forward our local environment to the Prometheus instance running in Kubernetes. To do this execute `kubectl port-forward service/prometheus-operated 9090:9090 --namespace prometheus` in your terminal. The **service** called **prometheus-operated** is created by the Operator for accessing the Prometheus instance you created. 

If you wish to see Kubernetes Services in the **prometheus** namespace, then execute `kubectl get services --namespace prometheus` in your terminal.

You will now be able to access Prometheus in your web browser at [http://localhost:9090](http://localhost:9090).

![Prometheus](/prometheus/deploying-prometheus/access-prometheus/images/prometheus.png?classes=shadow&width=55pc)