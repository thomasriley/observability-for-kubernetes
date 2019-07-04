---
title: "Node Exporter"
date: 2019-07-04T16:21:58+01:00
weight: 20
draft: false
---

## Overview

The Node Exporter is a Prometheus Exporter that owned by the Prometheus project. It is not specific to Kubernetes and is designed to expose hardware and OS metrics from *NIX based Kernels. The project can be found [here](https://github.com/prometheus/node_exporter) on GitHub.

We will look at using the Node Exporter to expose metrics for each node running in a Kubernetes cluster.

## Deployment

The Node Exporter needs to run on each node in the Kubernetes cluster therefore we will use a DaemonSet to acheive this.

Create a file called **node-exporter.yaml** and add the following:

```yaml
---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    app: node-exporter
  name: node-exporter
  namespace: prometheus
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
      labels:
        app: node-exporter
    spec:
      containers:
      - args:
        - --web.listen-address=0.0.0.0:9100
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        image: quay.io/prometheus/node-exporter:v0.18.1
        imagePullPolicy: IfNotPresent
        name: node-exporter
        ports:
        - containerPort: 9100
          hostPort: 9100
          name: metrics
          protocol: TCP
        resources:
          limits:
            cpu: 200m
            memory: 50Mi
          requests:
            cpu: 100m
            memory: 30Mi
        volumeMounts:
        - mountPath: /host/proc
          name: proc
          readOnly: true
        - mountPath: /host/sys
          name: sys
          readOnly: true
      hostNetwork: true
      hostPID: true
      restartPolicy: Always
      tolerations:
      - effect: NoSchedule
        operator: Exists
      - effect: NoExecute
        operator: Exists
      volumes:
      - hostPath:
          path: /proc
          type: ""
        name: proc
      - hostPath:
          path: /sys
          type: ""
        name: sys
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: node-exporter
  name: node-exporter
  namespace: prometheus
spec:
  ports:
  - name: node-exporter
    port: 9100
    protocol: TCP
    targetPort: 9100
  selector:
    app: node-exporter
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    app: node-exporter
    serviceMonitorSelector: prometheus
  name: node-exporter
  namespace: prometheus
spec:
  endpoints:
  - honorLabels: true
    interval: 30s
    path: /metrics
    targetPort: 9100
  jobLabel: node-exporter
  namespaceSelector:
    matchNames:
    - prometheus
  selector:
    matchLabels:
      app: node-exporter
```

The above YAML will create a DaemonSet that launches the Node Exporter on each node in the Kubernetes cluster. We then create a Kubernetes Service and ServiceMonitor to scrape metrics from all instances of Node Exporter.

Go ahead and install Node Exporter into your Kubernetes cluster by executing `kubectl apply -f node-exporter.yaml`.

You can then use `kubectl get pods --namespace prometheus` to see the **node-exporter** Pod(s) being created by Kubernetes. After a brief moment you can then check the configured Targets in Prometheus and you will see that **kube-state-metrics** is now being successfully scraped.

## Useful Metrics

Node Exporter has numerous collectors designed to gather OS and hardware metrics from various sources on a node. If you check the log output from a Node Exporter Pod using `kubectl logs` you can see the collectors that are active:

```shell
$kubectl logs node-exporter-c8cwp --namespace prometheus
time="2019-07-04T15:47:47Z" level=info msg="Enabled collectors:" source="node_exporter.go:97"
time="2019-07-04T15:47:47Z" level=info msg=" - arp" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - bcache" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - bonding" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - conntrack" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - cpu" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - cpufreq" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - diskstats" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - edac" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - entropy" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - filefd" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - filesystem" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - hwmon" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - infiniband" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - ipvs" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - loadavg" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - mdadm" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - meminfo" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - netclass" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - netdev" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - netstat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - nfs" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - nfsd" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - pressure" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - sockstat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - stat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - textfile" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - time" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - timex" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - uname" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - vmstat" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - xfs" source="node_exporter.go:104"
time="2019-07-04T15:47:47Z" level=info msg=" - zfs" source="node_exporter.go:104"
```

If you refer back to the Node Exporter [documenation](https://github.com/prometheus/node_exporter#collectors) you can see the metric that each of these collectors uses to acquire metrics. For example, the **arp** collector exposes the metrics available in **/proc/net/arp** on Linux.

In Prometheus, you will see that the majority of metrics exposed by the Node Exporter are prefixed with **node__**. For example, the arp collector described above exposes a metric called **node_arp_entries** that contains the number of ARP entries in the ARP table for each network interface on a node.
