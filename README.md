# Metrics visualisation PoC

## Description

This is a PoC for a single ArgoCD application to provide observability for a Kubernetes cluster. It is composed of Prometheus, Grafana (both provided by the kube-prometheus-stack chart), a customised version of k8s-sidecar, and custom metrics exporters. This solution will automatically pick up new metrics exporters in the mn namespace (if they have serviceMonitor.enabled) and will automatically display Grafana dashboards specified in its dashboards/kong.json file.

To start the app, click ‘New App’ in ArgoCD and point it to this Git repository. Most of the components are included as subchart dependencies in the Chart.yaml file. The exception is **k8s-sidecar**.

This is a helper app which watches a chosen set of ConfigMaps in a Kubernetes cluster and maintains a local copy of the files specified in those ConfigMaps. It can detect updates of ConfigMaps, and we have modified it to also detect deletions.

k8s-sidecar is a controller, providing a non-terminating control loop that regulates the state of a system. It tracks at least one Kubernetes resource type. These objects have a spec field that represents the desired state. The controller brings the current state come closer to that desired state. A Kubernetes cluster could be changing at any point as work happens and control loops automatically fix failures. This means that, potentially, it never reaches a stable state. As long as the controllers are active, it doesn't matter if the overall state is stable or not. The controller pattern generally looks like this:

![Oreilly kube controller](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/assets/prku_0102.png)

This PoC also demonstrates a declarative approach to specifying the Prometheus metrics exporters needed for the cluster. These are small apps, typically running in their own pods, which scrape metrics from an app such as a server or database, and expose it to Prometheus.

To add a metrics exporter, include it as a dependency in Chart.yaml, and pass any required values in values.yaml (e.g. the URL of the app to be scraped within the cluster, any required login details, and the flag to enable that exporter's ServiceMonitor which lets it be discovered by Prometheus).


## Modifications to k8s-sidecar

To make k8s-sidecar work without a shared volume with Grafana we had to modify the source code which was written in Python.  The way to do that was either change the code and re-build the docker image or mount the updated file in the pod.
We chose the second approach due to a lower technical debt. This solution requires less maintenance, no CI/CD Process for building the image and no storage for the docker image.
The ConfigMap we are using can be found in the following path:
```bash 
./k8s-sidecar/templates/configmap.yaml
```

Added labels for folderID and folderUid - for Grafana API in order to install the configmaps in the desired folders. One argument must be used, if both are supplied then folderId has priority.

Added support for DELETE, when the ConfigMap gets deleted it creates a request and the dashboard deletes as well from Grafana.

Basic overview of how it works in our case:

![Our diagram](diagram.jpg)

## Requirements:

 - a Kubernetes cluster

 - ArgoCD

 - some apps to scrape... currently this PoC is set up to scrape two instances each of Redis and Postgres

## Solution Components:

 - Prometheus

 - Grafana

 - k8s-sidecar

 - exporter-bootstrap

 - prometheus-redis-exporter - an example of an exporter which has a 1:1 relation with the app instance being scraped (so we deploytwo exporter instances, once for each Redis instance)

 - prometheus-postgres-exporter - an example of an exporter which can have a 1:many relation with app instances (so we deploy one exporter instance to scrape two separate Postgres instances)

## Workarounds and known issues

Prometheus admission webhooks have been disabled for ease of development, as they were consistently failing to be deleted in ArgoCD. prometheusOperator.tls has been disabled as part of this too.

For simplicity, Prometheus has been set to discover **all** metrics exporters in its namespace. This avoids the need to configure labels.

The Postgres exporter supports connecting to multiple instances, but **only** by using a Secret containing connection strings in a particular format. See code comment [here](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml). Note that **newlines are forbidden** in the connection string yet it is quite easy to introduce them accidentally during conversion to base64.

Moving dashboards is currently not supported, in order to move a folder you have to first delete the ConfigMap(dashboard) then apply it again with the right label.

## Configuration

Environment variables for k8s-sidecar:

`LABEL`

* description: Label that should be used for filtering
* required: true
* type: string

`FOLDER`

* description: Folder where the files should be placed
* required: true
* type: string

`FOLDER_ANNOTATION`

* description: The annotation the sidecar will look for in configmaps to override the destination folder for files, defaults to "k8s-sidecar-target-directory"
* required: false
* type: string

`NAMESPACE`

* description: If specified, the sidecar will search for config-maps inside this namespace. Otherwise the namespace in which the sidecar is running will be used. It's also possible to specify ALL to search in all namespaces.
* required: false
* type: string

`METHOD`

* description: If METHOD is set with LIST, the sidecar will just list config-maps and exit. Default is watch.
* required: false
* type: string

`REQ_URL`

* description: URL to which send a request after a configmap got reloaded
* required: false
* type: URI

`REQ_METHOD`

* description: Request method GET(default) or POST
* required: false
* type: string

`REQ_PAYLOAD`

* description: If you use POST you can also provide json payload
* required: false
* type: json

`REQ_RETRY_TOTAL`

* description: Total number of retries to allow
* required: false
* default: 5
* type: integer

`REQ_RETRY_CONNECT`

* description: How many connection-related errors to retry on
* required: false
* default: 5
* type: integer

`REQ_RETRY_READ`

* description: How many times to retry on read errors
* required: false
* default: 5
* type: integer

`REQ_RETRY_BACKOFF_FACTOR`

* description: A backoff factor to apply between attempts after the second try
* required: false
* default: 0.2
* type: float

`REQ_TIMEOUT`

* description: many seconds to wait for the server to send data before giving up
* required: false
* default: 10
* type: float

`SKIP_TLS_VERIFY`

* description: Set to true to skip tls verification for kube api calls
* required: false
* type: boolean

## Security

Login details for apps to be scraped, as well as for Grafana, are currently stored in plain text in values.yaml. This is suitable for a PoC but not for production usage.

The k8s-sidecar controller is secure as it only has limited privileges (read-only access to ConfigMaps) granted by the Role and RoleBinding resources. It follows the Kubernetes security implementation.

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ include "k8s-sidecar.fullname" . }}
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["configmaps", "secrets"]
  verbs: ["get", "watch", "list"]
