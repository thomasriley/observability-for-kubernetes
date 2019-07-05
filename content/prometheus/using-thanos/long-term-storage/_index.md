---
title: "Long Term Storage"
date: 2019-07-04T20:26:38+01:00
weight: 20
draft: false
---

## Long Term Storage 

Now that we have enabled high availability with Prometheus using Thanos we can look at the next killer feature of Thanos, long term storage of metrics!

To enable this, we first need to create a bucket on an object store such as AWS S3 or GCP Storage. Next we enable the Thanos Sidecar to upload metrics to the object store. Lastly, we need to deploy another Thanos component called Store that acts as an API for the metrics available in the object store and will be queries by the existing Thanos Query instance. The diagram below shows this.

![Thanos With Long Term Storage](/prometheus/using-thanos/long-term-storage/images/long-term-storage.png?classes=shadow&width=40pc)

From the perspective of users they are completely unaware that when they execute a Prometheus query Thanos is querying the for metrics from both the Prometheus instances and from the object storage. 

How does this work? Prometheus stores metrics in **blocks**. Intially, it uses a black in memory but periodically, typically every 2 hours, it writes the current in memory block out to the filesystem. As the Thanos Sidecar in the Prometheus Pod has the same shared filesystem as the Prometheus container it can see the new block that Prometheus writes to the filesystem and once it does the Thanos Sidecar uploads the block to the object storage as per its configuration. This is shown below in the diagram.

![Thanos Sidecar Upload](/prometheus/using-thanos/long-term-storage/images/thanos-sidecar-upload.png?classes=shadow&width=40pc)

Lets now implement this. For this example I created a Google Cloud Storage bucket called **observability-for-kubernetes-thanos-demo** and provisioned a Service Account with 'Storage Admin' permissions so that Thanos can read and write to the storage bucket.

We now need to create a Thanos Object Store configuration file that provides the bucket configuration and credentials. The structure of this file differs per cloud provider, you can see the [Thanos Documenation](https://github.com/improbable-eng/thanos/blob/master/docs/storage.md) to see the different options but for this example we will proceed using GCP.

We will create a Kubernetes Secret that contains the GCP Service Account and Bucket Name. The example below shows the expected structure, however the GCP Service Account is not valid for obvious reasons!

```yaml
type: GCS
config:
  bucket: "observability-for-kubernetes-thanos-demo"
  service_account: |-
  {
    "type": "service_account",
    "project_id": "kubernetes-cloud-lab",
    "private_key_id": "",
    "private_key": "-----BEGIN PRIVATE KEY-----\n\n-----END PRIVATE KEY-----\n",
    "client_email": "observability-for-kubernetes@kubernetes-cloud-lab.iam.gserviceaccount.com",
    "client_id": "",
    "auth_uri": "https://accounts.google.com/o/oauth2/auth",
    "token_uri": "https://oauth2.googleapis.com/token",
    "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
    "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/observability-for-kubernetes%40kubernetes-cloud-lab.iam.gserviceaccount.com"
  }
```

We should take the valid version of the example above and base64 encode it. You can use `base64` in your terminal for doing this. Then create a file called **thanos-object-secret.yaml** and include the Kubernetes Secret, as shown below:

```yaml
---
apiVersion: v1
kind: Secret
metadata:
  name: thanos-config
data:
  thanos.config: dHlwZTogR0NTCmNvbmZpZzoKICBidWNrZXQ6ICJvYnNlcnZhYmlsaXR5LWZvci1rdWJlcm5ldGVzLXRoYW5vcy1kZW1vIgogIHNlcnZpY2VfYWNjb3VudDogfC0KICB7CiAgICAidHlwZSI6ICJzZXJ2aWNlX2FjY291bnQiLAogICAgInByb2plY3RfaWQiOiAia3ViZXJuZXRlcy1jbG91ZC1sYWIiLAogICAgInByaXZhdGVfa2V5X2lkIjogIiIsCiAgICAicHJpdmF0ZV9rZXkiOiAiLS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tXG5cbi0tLS0tRU5EIFBSSVZBVEUgS0VZLS0tLS1cbiIsCiAgICAiY2xpZW50X2VtYWlsIjogIm9ic2VydmFiaWxpdHktZm9yLWt1YmVybmV0ZXNAa3ViZXJuZXRlcy1jbG91ZC1sYWIuaWFtLmdzZXJ2aWNlYWNjb3VudC5jb20iLAogICAgImNsaWVudF9pZCI6ICIiLAogICAgImF1dGhfdXJpIjogImh0dHBzOi8vYWNjb3VudHMuZ29vZ2xlLmNvbS9vL29hdXRoMi9hdXRoIiwKICAgICJ0b2tlbl91cmkiOiAiaHR0cHM6Ly9vYXV0aDIuZ29vZ2xlYXBpcy5jb20vdG9rZW4iLAogICAgImF1dGhfcHJvdmlkZXJfeDUwOV9jZXJ0X3VybCI6ICJodHRwczovL3d3dy5nb29nbGVhcGlzLmNvbS9vYXV0aDIvdjEvY2VydHMiLAogICAgImNsaWVudF94NTA5X2NlcnRfdXJsIjogImh0dHBzOi8vd3d3Lmdvb2dsZWFwaXMuY29tL3JvYm90L3YxL21ldGFkYXRhL3g1MDkvb2JzZXJ2YWJpbGl0eS1mb3Ita3ViZXJuZXRlcyU0MGt1YmVybmV0ZXMtY2xvdWQtbGFiLmlhbS5nc2VydmljZWFjY291bnQuY29tIgogIH0=
```

The value of **thanos.config** should be set to the base64 string that was just generated.

Apply **thanos-object-secret.yaml** to Kubernetes by executing `kubectl apply -f thanos-object-secret.yaml`:

```shell
$kubectl apply -f thanos-object-secret.yaml
secret/cluster-thanos-config created
```

With the object store configuration deployed to Kubernetes, next update the Prometheus resource adding the **objectStorageConfig** key so that the Prometheus Operator configures the Thanos Sidecare with the object storage config that has just been deployed to Kubernetes.

The Prometheus resource should look similar to the below with this added:

```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: prometheus
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - prometheus
          topologyKey: kubernetes.io/hostname
  baseImage: quay.io/prometheus/prometheus
  logLevel: info
  podMetadata:
    annotations:
      cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
    labels:
      app: prometheus
      thanos-store-api: "true"
  replicas: 2
  thanos:
    version: v0.4.0
    resources:
      limits:
        cpu: 500m
        memory: 500Mi
      requests:
        cpu: 100m
        memory: 500Mi
    objectStorageConfig:
      key: thanos.config
      name: thanos-config
  resources:
    limits:
      cpu: 1
      memory: 2Gi
    requests:
      cpu: 1
      memory: 2Gi
  retention: 12h
  serviceAccountName: prometheus-service-account
  serviceMonitorSelector:
    matchLabels:
      serviceMonitorSelector: prometheus
  externalLabels:
    cluster_environment: workshop
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
```

Edit the **prometheus.yaml** file we created previously to reflect the changes above. Under the **objectStorageConfig** key set the **name** to be the name of the Kubernetes Secret we just created and the **key** to be the name of the secret key we used, which in this case is **thanos.config**.

Apply the updated **prometheus.yaml** to Kubernetes by running `kubectl apply -f prometheus.yaml`. To see an example of the full YAML file that reflects the changes described above see [here](/prometheus/using-thanos/long-term-storage/static/thanos-with-object-config.yaml).

Once the Prometheus Pods have restarted with the new configuration, use `kubectl logs` to view the logs from the **thanos-sidecar** container in one of the Prometheus Pods, as shown below:

```shell
$kubectl logs prometheus-prometheus-0 thanos-sidecar
level=info ts=2019-07-05T20:48:04.403642465Z caller=flags.go:87 msg="gossip is disabled"
level=info ts=2019-07-05T20:48:04.403893193Z caller=main.go:257 component=sidecar msg="disabled TLS, key and cert must be set to enable"
level=info ts=2019-07-05T20:48:04.403923795Z caller=factory.go:39 msg="loading bucket configuration"
level=info ts=2019-07-05T20:48:04.404486705Z caller=sidecar.go:319 msg="starting sidecar" peer="no gossip"
level=info ts=2019-07-05T20:48:04.404636222Z caller=main.go:309 msg="Listening for metrics" address=0.0.0.0:10902
level=info ts=2019-07-05T20:48:04.404674065Z caller=reloader.go:154 component=reloader msg="started watching config file and non-recursively rule dirs for changes" cfg= out= dirs=
level=info ts=2019-07-05T20:48:04.40474036Z caller=sidecar.go:260 component=sidecar msg="Listening for StoreAPI gRPC" address=[10.8.2.14]:10901
level=info ts=2019-07-05T20:48:06.414276291Z caller=sidecar.go:176 msg="successfully loaded prometheus external labels" external_labels="{cluster_environment=\"workshop\",prometheus=\"prometheus/prometheus\",prometheus_replica=\"prometheus-prometheus-0\"}"
level=info ts=2019-07-05T20:48:34.441923855Z caller=shipper.go:350 msg="upload new block" id=01DF1NSPGCHYM8T82R8BKAX4HP
level=info ts=2019-07-05T20:48:35.12152462Z caller=shipper.go:350 msg="upload new block" id=01DF1R3VHXY2NA2ZH4W4TVM33H
level=info ts=2019-07-05T21:00:04.457140171Z caller=shipper.go:350 msg="upload new block" id=01DF1YZJT0GNZ2G0YWJWJ3NE25
```

If your instances of Prometheus have been running long enough (atleast 2 hours) there should be a block on the filesystem ready to be uploaded. If there is, you will see it upload the block to the storage bucket as shown above. See the log entry **"upload new block"** where it begins the upload. If you check the bucket in the cloud provider console you will the blocks are now present.

## Thanos Components

Now that the Thanos Sidecar is uploading blocks to the object store, we need to deploy two addtional components: Thanos Store and Thanos Compact.

### Thanos Store

Thanos Store acts as an API for querying Prometheus metrics stored in the object store.

Update **prometheus.yaml** to also include the following:

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-store
  labels:
    app: thanos-store
    thanos-store-api: "true"
spec:
  replicas: 1
  serviceName: thanos-store
  selector:
    matchLabels:
      app: thanos-store
      thanos-store-api: "true"
  template:
    metadata:
      labels:
        app: thanos-store
        thanos-store-api: "true"
    spec:
      containers:
      - name: thanos-store
        image: improbable/thanos:v0.5.0
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 1Gi
        args:
        - "store"
        - "--data-dir=/prometheus/cache"
        - "--objstore.config-file=/config/thanos.config"
        - "--log.level=info"
        - "--index-cache-size=256MB"
        - "--chunk-pool-size=256MB"
        - "--store.grpc.series-max-concurrency=30"
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        - name: cluster
          containerPort: 10900
        volumeMounts:
        - mountPath: /prometheus
          name:  thanos-store-storage
        - mountPath: /config/
          name: thanos-config
      volumes:
      - name: thanos-config
        secret:
          secretName: thanos-config
  volumeClaimTemplates:
  - metadata:
      name: thanos-store-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Apply this to Kubernetes by running `kubectl apply -f prometheus.yaml`.

Kubernetes will launch a single Thanos Store pod as per the configuration above. If you check the available **Stores** in the Thanos Query UI, you will now see Thanos Store listed in addition to the two Thanos Sidecars running alongside Prometheus. Now when querying via Thanos Query, the query will execute across both the Prometheus instances and also the metrics stored in the object store!

![Thanos Query Stores](/prometheus/using-thanos/long-term-storage/images/thanos-query-with-store.png?classes=shadow&width=40pc)

### Thanos Compact

Thanos Compact is the final Thanos component we need to deploy. Compact performs three main tasks:

* It executes a Prometheus compaction job on the blocks in the object store. Typically Prometheus would execute this on blocks locally on the filesystem but the process is disabled by Prometheus Operator when using Thanos.
* It also executes a downsampling process on metrics in the object store. Prometheus stores metrics with a resolution of 1 minute. However, if you were to execute a query over a period of months or years on a Prometheus environment using Thanos for long term storage, the number of datapoints returned would be excessive! Therefore, Compact performs the downsampling job adding a 5 minute and 1 hour sample in addition to the 1 minute sample by creating a new block and discarding the original once downsampling is completed. When executing queries, Thanos will automatically select the most appropriate sample to use.
* Lastly, it is also possible to set retention periods for the 1 minute, 5 minute and 1 hour samples. Compact will apply these retentions if they are set.

Now lets deploy Thanos Compact. Update **prometheus.yaml** to also include the following:

```yaml
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: thanos-compact
  labels:
    app: thanos-compact
spec:
  replicas: 1
  serviceName: thanos-compact
  selector:
    matchLabels:
      app: thanos-compact
  template:
    metadata:
      labels:
        app: thanos-compact
    spec:
      containers:
      - name: thanos-compact
        image: improbable/thanos:v0.5.0
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 1Gi
        args:
        - "compact"
        - "--data-dir=/prometheus/compact"
        - "--objstore.config-file=/config/thanos.config"
        - "--log.level=info"
        - "--retention.resolution-raw=2d"
        - "--retention.resolution-5m=5d"
        - "--retention.resolution-1h=10d"
        - "--consistency-delay=15m"
        - "--wait"
        ports:
        - name: http
          containerPort: 10902
        - name: grpc
          containerPort: 10901
        - name: cluster
          containerPort: 10900
        volumeMounts:
        - mountPath: /prometheus
          name: thanos-compact-storage
        - mountPath: /config/
          name: thanos-config
      volumes:
      - name: thanos-config
        secret:
          secretName: thanos-config
  volumeClaimTemplates:
  - metadata:
      name: thanos-compact-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

Apply this to Kubernetes by running `kubectl apply -f prometheus.yaml`.

Kubernetes will launch a single Thanos Compact Pod which will, once running, start performing the actions described above on blocks stored in the object store.

To see an example of the full YAML file that reflects the changes described above with Thanos Store and Compact see [here](/prometheus/using-thanos/long-term-storage/static/thanos-with-components.yaml).
