---
title: "Launch Prometheus Instance"
date: 2019-07-03T10:35:40+01:00
weight: 20
draft: false
---

### Launching Prometheus 

Now that we have deployed Prometheus Operator we can use it to launch an instance of Prometheus.

When we deployed the Operator, one of the tasks it performed when it first launched was to is install a number Custom Resource Definitions (CRDs) into Kubernetes. Out of the box, Kubernetes ships with many powerful Controllers such as the Deployment or Statefulset. CRDs provide a method of building completely bespoke Controllers that provide logic to a specific function. In this case, the CRDs installed by Prometheus Operator provide a means for launching and configuring Prometheus within Kubernetes.

If you run the `kubectl get customresourcedefinitions` command in your terminal you will see four CRDs provided by the Operator:

```shell
$kubectl get customresourcedefinitions
NAME                                           CREATED AT
alertmanagers.monitoring.coreos.com            2019-07-02T13:13:21Z
prometheuses.monitoring.coreos.com             2019-07-02T13:13:21Z
prometheusrules.monitoring.coreos.com          2019-07-02T13:13:21Z
servicemonitors.monitoring.coreos.com          2019-07-02T13:13:21Z
```

To begin with we will be making use of the **prometheuses.monitoring.coreos.com** Custom Resource Definition.

Create a file called **prometheus.yaml** and add the following:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
spec:
  baseImage: quay.io/prometheus/prometheus
  logLevel: info
  podMetadata:
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    labels:
      app: prometheus
  replicas: 1
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
  retention: 12h
  serviceAccountName: prometheus-service-account
  storage:
    volumeClaimTemplate:
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: prometheus-pvc
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
  version: v2.10.0
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: "prometheus-service-account"
  namespace: "prometheus"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: "prometheus-cluster-role"
rules:
- apiGroups:
  - ""
  resources:
  - nodes
  - services
  - endpoints
  - pods
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/metrics
  verbs:
  - get
- nonResourceURLs:
  - "/metrics"
  verbs:
  - get
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: "prometheus-cluster-role-binding"
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: "prometheus-cluster-role"
subjects:
- kind: ServiceAccount
  name: "prometheus-service-account"
  namespace: prometheus
```

Now apply this YAML to your Kubernetes cluster using `kubectl apply -f prometheus.yaml`. Kubectl will show that it has successfully created the configuration, as shown below:

```shell
$kubectl apply -f prometheus.yaml
prometheus.monitoring.coreos.com/prometheus created
serviceaccount/prometheus-service-account created
clusterrole.rbac.authorization.k8s.io/prometheus-cluster-role created
clusterrolebinding.rbac.authorization.k8s.io/prometheus-cluster-role-binding created
```

Success! If you now list Pods in the **prometheus** namespace using `kubectl get pods --namespace prometheus` you will see an instance of Prometheus running alongside the Prometheus Operator:

```shell
$kubectl get pods
NAME                                            READY   STATUS    RESTARTS   AGE
prometheus-operator-operator-86bc4d5568-shs69   1/1     Running   0          10m
prometheus-prometheus-0                         3/3     Running   0          1m
```

If you now check for the custom Prometheus resource that you have just installed into the cluster using `kubectl get prometheus --namespace prometheus` you will see the single result named **prometheus**:

```shell
$kubectl get prometheus --namespace prometheus
NAME         AGE
prometheus   6m
```

The Prometheus Operator acts as a Controller for the Custom Resources. When you deployed the **Prometheus** resource the Operator created the Prometheus instance, that you just identified when getting a list Pods in the **prometheus** namespace.

### What does this mean?

Lets now take a look at the **prometheus.yaml** file we applied to Kubernetes and see what each section means.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
```

Here we define that we wish to create an object called **prometheus** that is of the type **Prometheus** as defined by the Kind and that this Kind is part of the API Version **monitoring.coreos.com/v1**, that was previously installed into Kubernetes by the Prometheus Operator as a Custom Resource Definition. This object will be created in the **prometheus** namespace.

Everything then under the **spec** of the YAML file defines what the instance of Prometheus should look like.

```yaml
  replicas: 1
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
```

Here we define the Resource limits (CPU & Memory) that each Prometheus Pod will be granted within Kubernetes. We also specify the number of instances that we require by setting **replicas**, in this example we have just the 1 instance.

```yaml
  baseImage: quay.io/prometheus/prometheus
  version: v2.10.0
```

Setting the **baseImage** defines the actual Prometheus Docker image to be used. This will actually be defaulted to the Docker image that is released by the Prometheus project however we included it as an example. The **version** field sets the version of Prometheus you wish to use. You can see available versions on the [GitHub project](https://github.com/prometheus/prometheus/releases).

```yaml
  storage:
    volumeClaimTemplate:
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: prometheus-pvc
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi
```

This block defines the storage that will be used by Prometheus. By default the Operator will create Prometheus Pods that use local storage only by using an [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir). If you wish to retain the state of Prometheus and therefore the metrics that it stores when re-launching Prometheus, such as during a version upgrade, then you need to use persistent storage. The **PersistentVolumeClaim (PVC)** defines the specification of the storage to be used by Prometheus. In this example we are creating a persistent disk that is 10Gi for each instance of Prometheus that is created.

In your terminal if you execute `kubectl get persistentvolumeclaim --namespace prometheus` you will see the PVC that has been created and Bound to a Persistent Volume for the single instance of Prometheus that you have created:

```shell
$kubectl get persistentvolumeclaim --namespace prometheus
NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
prometheus-pvc-prometheus-prometheus-0   Bound    pvc-c85b2a3b-9d7a-11e9-9e3c-42010a840079   10Gi       RWO            standard       21m
```
The **ServiceAccount**, **ClusterRoleBinding** and **ClusterRoleBinding** are required for providing the Prometheus Pod with the required permissions to access the Kubernetes API as part of its service discovery process.

Lastly lets look at some of the Prometheus specific configuration of the **prometheus.yaml** file:

```yaml
  logLevel: info
  retention: 12h
```

Here we define that Prometheus should retain 12 hours of metrics and that it should log using the **info** log level.

The Prometheus Operator GitHub project provides a [full set](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md) of API documentation that defines the fields that can be set on the CRDs that it provides. You can see the specification for the **Prometheus** **Kind** [here](https://github.com/coreos/prometheus-operator/blob/master/Documentation/api.md#prometheusspec).