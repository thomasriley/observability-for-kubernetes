---
title: "Deploying Prometheus Operator with Helm"
date: 2019-07-02T12:08:47+01:00
weight: 10
draft: false
---

First we will use the community maintained [Helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator) for deploying Prometheus Operator to Kubernetes. Out of the box, the Helm chart will also configure the operator install an instance of Prometheus however to begin with lets deploy a standalone instance of the Operator.

Create a file called **values.yaml** containing the following:

```yaml
defaultRules:
  create: false
alertmanager:
  enabled: false
grafana:
  enabled: false
kubeApiServer:
  enabled: false
kubelet:
  enabled: false
kubeControllerManager:
  enabled: false
coreDns:
  enabled: false
kubeEtcd:
  enabled: false
kubeScheduler:
  enabled: false
kubeStateMetrics:
  enabled: false
nodeExporter:
  enabled: false
prometheus:
  enabled: false
```

Then install the Prometheus Operator via Helm using the **helm upgrade** command as shown below:

```shell
helm upgrade --install prometheus-operator stable/prometheus-operator --namespace prometheus --values values.yaml
```

When this executes, Helm will display all of the resources it has successfully created in Kubernetes:

```shell
$ helm upgrade --install prometheus-operator stable/prometheus-operator --namespace prometheus --values values.yaml

Release "prometheus-operator" does not exist. Installing it now.
NAME:   prometheus-operator
LAST DEPLOYED: Tue Jun 25 22:06:52 2019
NAMESPACE: prometheus
STATUS: DEPLOYED

RESOURCES:
==> v1/ClusterRole
NAME                              AGE
prometheus-operator-operator      1s
prometheus-operator-operator-psp  1s

==> v1/ClusterRoleBinding
NAME                              AGE
prometheus-operator-operator      1s
prometheus-operator-operator-psp  1s

==> v1/Deployment
NAME                          READY  UP-TO-DATE  AVAILABLE  AGE
prometheus-operator-operator  0/1    1           0          1s

==> v1/Pod(related)
NAME                                           READY  STATUS             RESTARTS  AGE
prometheus-operator-operator-694f88774b-q4r64  0/1    ContainerCreating  0         1s

==> v1/Service
NAME                          TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)   AGE
prometheus-operator-operator  ClusterIP  10.11.250.245  <none>       8080/TCP  1s

==> v1/ServiceAccount
NAME                          SECRETS  AGE
prometheus-operator-operator  1        1s

==> v1/ServiceMonitor
NAME                          AGE
prometheus-operator-operator  1s

==> v1beta1/PodSecurityPolicy
NAME                          PRIV   CAPS      SELINUX   RUNASUSER  FSGROUP    SUPGROUP  READONLYROOTFS  VOLUMES
prometheus-operator-operator  false  RunAsAny  RunAsAny  MustRunAs  MustRunAs  false     configMap,emptyDir,projected,secret,downwardAPI,persistentVolumeClaim


NOTES:
The Prometheus Operator has been installed. Check its status by running:
  kubectl --namespace prometheus get pods -l "release=prometheus-operator"

Visit https://github.com/coreos/prometheus-operator for instructions on how
to create & configure Alertmanager and Prometheus instances using the Operator.

```

Above you can see that Helm has deployed the **stable/prometheus-operator** Helm chart under the release name **prometheus-operator** into the Kubernetes namespace **prometheus** using the Helm values we created above in values.yaml.

If you then use Kubectl to list the Pods in the **prometheus** namespace you will see the Prometheus Operator is now installed:

```shell
$ kubectl get pods -n prometheus
NAME                                            READY   STATUS    RESTARTS   AGE
prometheus-operator-operator-694f88774b-q4r64   1/1     Running   0          6m47s
```
