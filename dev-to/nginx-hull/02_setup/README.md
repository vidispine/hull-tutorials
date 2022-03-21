## Introduction

This article series will go step by step through building a Helm Chart based on the HULL library. To allow a good comparison with the traditional Helm chart creation process and for educational purposes the goal is to rewrite the `ingress-nginx` Helm chart in it's latest version that exists at time of writing ([which is version 4.0.6](https://artifacthub.io/packages/helm/ingress-nginx/ingress-nginx/4.0.6)). 

You may ask why the `ingress-nginx` Helm chart? For a number of reasons this is an interesing topic for this tutorial: 
- the `nginx-ingress` Helm chart is most likely one of the most frequently downloaded and used Helm chart around the globe. Many people interact with it daily and may have a deeper understanding of its structure
- it has a medium to high complexity which makes the goal of translating it to a HULL based chart not trivial and has the power of showcasing the real-life applicability of the HULL library to Helm chart creation 
- lastly, a deep dive into such a more complex Helm Chart may still offer interesting insights to you if you are new-ish to understanding Helm chart mechanics or want to get to learn the `ingress-nginx`'s particular charts architecture better.

## Getting Helm

Before you can get started you need to make sure you have a [fresh Helm version installed](https://helm.sh/docs/intro/install/). Then open up a Linux terminal of your choice and when you type in: 


```bash
$ helm version
```

the output should be similar to this:

```yaml
version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"clean", GoVersion:"go1.16.5"}
```

Now you are good to go!

## Creating the test project

First switch to your working directory, eg. `home`: 

```bash
$ cd ~
```

and there create a directory for our first tutorial part and switch to it:

```bash
$ mkdir 02_setup && cd 02_setup
```
 
## Getting the original `ingress-nginx` chart

For reference and analysis you can download a copy of the original  `ingress-nginx` helm chart and extract it like this:

```bash
$ curl -L https://github.com/kubernetes/ingress-nginx/releases/download/helm-chart-4.0.6/ingress-nginx-4.0.6.tgz -o nginx.tgz && tar -xvzf nginx.tgz -C . && rm nginx.tgz
```

To check if it was succesful list the `ingress-nginx` folder:

```bash
$ ls -R ingress-nginx
```

and you should see all the extracted files:

```yaml
ingress-nginx:
CHANGELOG.md  Chart.yaml  OWNERS  README.md  ci  templates  values.yaml

ingress-nginx/ci:
controller-custom-ingressclass-flags.yaml         deployment-autoscaling-values.yaml
daemonset-customconfig-values.yaml                deployment-customconfig-values.yaml
daemonset-customnodeport-values.yaml              deployment-customnodeport-values.yaml
daemonset-headers-values.yaml                     deployment-default-values.yaml
daemonset-internal-lb-values.yaml                 deployment-headers-values.yaml
daemonset-nodeport-values.yaml                    deployment-internal-lb-values.yaml
daemonset-podannotations-values.yaml              deployment-metrics-values.yaml
daemonset-tcp-udp-configMapNamespace-values.yaml  deployment-nodeport-values.yaml
daemonset-tcp-udp-values.yaml                     deployment-podannotations-values.yaml
daemonset-tcp-values.yaml                         deployment-psp-values.yaml
deamonset-default-values.yaml                     deployment-tcp-udp-configMapNamespace-values.yaml
deamonset-metrics-values.yaml                     deployment-tcp-udp-values.yaml
deamonset-psp-values.yaml                         deployment-tcp-values.yaml
deamonset-webhook-and-psp-values.yaml             deployment-webhook-and-psp-values.yaml
deamonset-webhook-values.yaml                     deployment-webhook-resources-values.yaml
deployment-autoscaling-behavior-values.yaml       deployment-webhook-values.yaml

ingress-nginx/templates:
NOTES.txt                               controller-hpa.yaml                  controller-serviceaccount.yaml
_helpers.tpl                            controller-ingressclass.yaml         controller-servicemonitor.yaml
admission-webhooks                      controller-keda.yaml                 default-backend-deployment.yaml
clusterrole.yaml                        controller-poddisruptionbudget.yaml  default-backend-hpa.yaml
clusterrolebinding.yaml                 controller-prometheusrules.yaml      default-backend-poddisruptionbudget.yaml
controller-configmap-addheaders.yaml    controller-psp.yaml                  default-backend-psp.yaml
controller-configmap-proxyheaders.yaml  controller-role.yaml                 default-backend-role.yaml
controller-configmap-tcp.yaml           controller-rolebinding.yaml          default-backend-rolebinding.yaml
controller-configmap-udp.yaml           controller-service-internal.yaml     default-backend-service.yaml
controller-configmap.yaml               controller-service-metrics.yaml      default-backend-serviceaccount.yaml
controller-daemonset.yaml               controller-service-webhook.yaml      dh-param-secret.yaml
controller-deployment.yaml              controller-service.yaml

ingress-nginx/templates/admission-webhooks:
job-patch  validating-webhook.yaml

ingress-nginx/templates/admission-webhooks/job-patch:
clusterrole.yaml         job-createSecret.yaml  psp.yaml   rolebinding.yaml
clusterrolebinding.yaml  job-patchWebhook.yaml  role.yaml  serviceaccount.yaml
```

## Setting up a HULL Chart

Good, now proceed by creating a new empty HULL based Helm chart.  [The steps are documented here](https://github.com/vidispine/hull/blob/main/hull/doc/setup.md) 
but you will create it from scratch here to understand what is needed. 

One way to get started is by using the `helm create` command to create the folder structure for an empty Helm chart. Run this command to create our `ingress-nginx-hull` Helm chart:

```bash
$ helm create ingress-nginx-hull
```

should produce this answer: 

```yaml
Creating ingress-nginx-hull
```

That should have created a folder `~/ingress-nginx-hull` with the usual required and example Helm chart files in there. Switch to the folder and examine the contents:

```bash
$ cd ingress-nginx-hull
```
```bash
$ ls -R
```
```yaml
.:
Chart.yaml  charts  templates  values.yaml

./charts:

./templates:
NOTES.txt  _helpers.tpl  deployment.yaml  hpa.yaml  ingress.yaml  service.yaml  serviceaccount.yaml  tests

./templates/tests:
test-connection.yaml
```

Delete examples and tests in the `/templates` folder, nothing besides the `hull.yaml` is needed:

```bash
$ rm -rf templates/*
```
```bash
$ ls -R
```
```yaml
.:
Chart.yaml  charts  templates  values.yaml

./charts:

./templates:
```

Additionally empty the `values.yaml` with the example data:

```bash
$ echo '' > values.yaml
```

Now edit the mandatory `Chart.yaml` which is updated to reflect the original charts information. Furthermore add the HULL library as requirement which is necessary to be able to use HULL.

### Choosing a HULL version

When choosing a HULL version for this, it is recommended that HULLs `major.minor` version must at least match the `kubeVersion` field otherwise the API version of objects created by HULL may be yet unknown to the clusters API version. Each HULL release branch is aligned with a specific Kubernetes version such as `1.20`, `1.21`, `1.22` and so on. The objects that HULL renders out will always be of the latest API version of the given object type that matches the HULL librarys version. So when an API is upgraded from beta to stable in a Kubernetes version jump, the latter HULL version will create objects with the stable API version and the former will create the beta API versioned objects. It is recommended to leave the `kubeVersion` set at `1.20.0` for this example so you will be able to work against any Kubernetes cluster with a version of `1.20.0` or above.

Replace `1.22.11` below with the latest available HULL version (more on versioning in a later article) and match the `kubeVersion` accordingly:

```bash
$ echo 'apiVersion: v2
appVersion: 1.0.4
description: Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer (built using HULL)
home: https://github.com/kubernetes/ingress-nginx
icon: https://upload.wikimedia.org/wikipedia/commons/thumb/c/c5/Nginx_logo.svg/500px-Nginx_logo.svg.png
keywords:
- ingress
- nginx
kubeVersion: ">=1.20.0-0"
maintainers:
- name: Me
name: ingress-nginx-hull
sources:
- https://github.com/kubernetes/ingress-nginx
type: application
version: 4.0.6
dependencies:
- name: hull
  version: "1.22.11"
  repository: "https://vidispine.github.io/hull"' > Chart.yaml
```

Verify it was written correctly:
```bash
$ cat Chart.yaml
```

returns:

```yaml
apiVersion: v2
appVersion: 1.0.4
description: Ingress controller for Kubernetes using NGINX as a reverse proxy and load balancer (built using HULL)
home: https://github.com/kubernetes/ingress-nginx
icon: https://upload.wikimedia.org/wikipedia/commons/thumb/c/c5/Nginx_logo.svg/500px-Nginx_logo.svg.png
keywords:
- ingress
- nginx
kubeVersion: ">=1.20.0-0"
maintainers:
- name: Me
name: ingress-nginx-hull
sources:
- https://github.com/kubernetes/ingress-nginx
type: application
version: 4.0.6
dependencies:
- name: hull
  version: "1.22.10"
  repository: "https://vidispine.github.io/hull"
```

Before starting work on the `values.yaml` you should download the HULL library by the Helm way of updating Chart dependencies:

```bash
$ helm dependency update
```

to download the HULL chart from the GitHub hosted HelmChart repository:
```yaml
Getting updates for unmanaged Helm repositories...
...Successfully got an update from the "https://vidispine.github.io/hull" chart repository
Saving 1 charts
Downloading hull from repo https://vidispine.github.io/hull
Deleting outdated charts
```

Great, one tiny thing left before you can start defining the objects of our `ingress-nginx-hull` Chart: copy over the `hull.yaml` from the now locally downloaded subchart to the main charts template folder. Without this step, nothing can be rendered via HULL since the `hull.yaml` contains the logic to convert `values.yaml` object definitions to rendered YAML files. We can do this with a one-liner where we extract the packaged HULL tgz file from the `/charts` directory to a local `temp` directory, copy over the `hull.yaml` and clean up the temp directory afterwards:

```bash
$ mkdir _tmp && tar -xvzf charts/hull-1.22.11.tgz -C _tmp && cp _tmp/hull/hull.yaml templates/hull.yaml && rm -rf _tmp
```

prints out the extracted files:

```yaml
hull/Chart.yaml
hull/Chart.lock
hull/values.yaml
hull/values.schema.json
hull/templates/_metadata.tpl
hull/templates/_metadata_annotations.tpl
hull/templates/_metadata_chartref.tpl
hull/templates/_metadata_fullname.tpl
hull/templates/_metadata_header.tpl
hull/templates/_metadata_labels.tpl
hull/templates/_metadata_name.tpl
hull/templates/_objects_base_plain.tpl
hull/templates/_objects_base_pod.tpl
hull/templates/_objects_base_rbac.tpl
hull/templates/_objects_base_spec.tpl
hull/templates/_objects_base_webhook.tpl
hull/templates/_objects_configmap.tpl
hull/templates/_objects_cronjob.tpl
hull/templates/_objects_horizontalpodautoscaler.tpl
hull/templates/_objects_ingress.tpl
hull/templates/_objects_pod.tpl
hull/templates/_objects_pod_container.tpl
hull/templates/_objects_pod_volume.tpl
hull/templates/_objects_registry.tpl
hull/templates/_objects_secret.tpl
hull/templates/_objects_service.tpl
hull/templates/_util.tpl
hull/templates/_util_transformations.tpl
hull/.helmignore
hull/LICENSE
hull/README.md
hull/doc/architecture.md
hull/doc/development.md
hull/doc/json_schema_validation.md
hull/doc/objects_base.md
hull/doc/objects_configmaps_secrets.md
hull/doc/objects_customresource.md
hull/doc/objects_horizontalpodautoscaler.md
hull/doc/objects_ingress.md
hull/doc/objects_pod.md
hull/doc/objects_registry.md
hull/doc/objects_service.md
hull/doc/objects_webhook.md
hull/doc/setup.md
hull/doc/transformations.md
hull/files/scripts/HullChart.ps1
hull/files/scripts/SetChartVersion.ps1
hull/files/scripts/SetYamlValues.ps1
hull/hull.yaml
```

Now check if the `hull.yaml` was copied correctly to the `templates` folder:
```bash
$ ls templates
```

should show:

```yaml
hull.yaml
```

### Inspecting default RBAC settings

Fantastic, all in place! If everything went right up to here you can take a first look at what HULL does by default in an otherwise unconfigured Helm chart. For this you can have Helm render out the YAMLs to `stdout` where you can study them instead of directly deploying objects to a cluster. `helm template` is the command used for this rendering out:

```bash
$ helm template .
```
```yaml
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-default
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-default
rules: []
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: release-name-ingress-nginx-hull-default
subjects:
- kind: ServiceAccount
  name: release-name-ingress-nginx-hull-default
```

There is already a lot you can see from this output. One thing you may ask is: why are there already objects created while I haven't even defined anything yet? That's because by default, HULL will render a 'default' set of RBAC objects which you can use and fine-tune for your application. The objects that are automatically created are:
- a default ServiceAccount with name `<release-name>-<chart-name>-default` which is preconfigured for all pods as the `serviceAccountName` to use
- a default Role with name `<release-name>-<chart-name>-default` which initially has no rules but you can add them as needed 
- a default RoleBinding with name `<release-name>-<chart-name>-default` that connects the `<release-name>-<chart-name>-default` Role with the `<release-name>-<chart-name>-default` ServiceAccount

This is like a RBAC bootstrap you can build upon easily but you may have different needs so if this setup does not fit your usecase, you can disable this default set of RBAC objects and specify different RBAC objects more tailored to your needs. 

Additionally, while using RBAC is the default with HULL you can also optionally turn off the rendering of RBAC objects all together if that is what you want (of course not recommended :) ).

### Inspecting the autogenerated metadata

Another standout aspect here is the autogenerated metadata for all objects which adheres to the [best practice Kubernetes metadata definition](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) and [Helm's recommendation on the subject matter](https://helm.sh/docs/chart_best_practices/labels/#standard-labels). Using HULL you get this default metadata created without you having to do anything and on top of this it is easy to [enrich all objects or objects of a particular type or just individual objects with metadata even further](https://github.com/vidispine/hull/tree/main/hull#advanced-example). 

## Wrap up
That's it for Part 2, in Part 3 looks at ConfigMaps and Secrets and you will add your first custom objects to the `ingress-nginx-hull` Helm chart! You can check out the complete code created so far as the outcome of this Part 2 [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/02_setup).

Thanks for reading!