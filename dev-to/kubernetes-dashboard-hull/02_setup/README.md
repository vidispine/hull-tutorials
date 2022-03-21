## Introduction

This article series will go step by step through building a Helm Chart based on the HULL library. To allow a good comparison with the traditional Helm chart creation process and for educational purposes the goal is to rewrite the `kubernetes-dashboard` Helm chart in it's latest version that exists at time of writing ([which is version 5.2.0](https://artifacthub.io/packages/helm/k8s-dashboard/kubernetes-dashboard/5.2.0)).

You may ask why the `kubernetes-dashboard` Helm chart? For a number of reasons this is an interesing topic for this tutorial:
- the `kubernetes-dashboard` Helm chart is most likely one of the most frequently downloaded and used Helm chart around the globe. Many people interact with it daily and may have a deeper understanding of its structure
- it has a medium complexity which makes the goal of translating it to a HULL based chart on the one hand has the power of showcasing the real-life applicability of the HULL library to Helm chart creation and on the other hand is doable with a reasonable amount of effort
- lastly, a deep dive into such a commonly used Helm Chart may still offer interesting insights to you if you are new-ish to understanding Helm chart mechanics or want to get to learn the `kubernetes-dashboard`'s particular charts architecture better

## Getting Helm

Before you can get started you need to make sure you have a [fresh Helm version installed](https://helm.sh/docs/intro/install/). Then open up a Linux terminal of your choice and when you type in:


```sh
helm version
```

the output should be similar to this:

```yml
version.BuildInfo{Version:"v3.6.3", GitCommit:"d506314abfb5d21419df8c7e7e68012379db2354", GitTreeState:"clean", GoVersion:"go1.16.5"}
```

Now you are good to go!

## Creating the test project

First switch to your working directory, eg. `home`: 

```sh
cd ~
```

and there create a directory for our project and first tutorial part and switch to it:

```sh
mkdir kubernetes-dashboard-hull
mkdir kubernetes-dashboard-hull/02_setup && cd kubernetes-dashboard-hull/02_setup
```
 
## Getting the original `kubernetes-dashboard` chart

For reference and analysis you can download a copy of the original  `kubernetes-dashboard` helm chart and extract it like this:

```sh
curl -L https://kubernetes.github.io/dashboard/kubernetes-dashboard-5.2.0.tgz -o kubernetes-dashboard.tgz && tar -xvzf kubernetes-dashboard.tgz -C . && rm kubernetes-dashboard.tgz
```

To check if it was succesful list the `kubernetes-dashboard` folder:

```sh
ls -R kubernetes-dashboard
```

and you should see all the extracted files:

```yml
kubernetes-dashboard:
Chart.lock  Chart.yaml  README.md  charts  templates  values.yaml

kubernetes-dashboard/charts:
metrics-server

kubernetes-dashboard/charts/metrics-server:
Chart.yaml  README.md  ci  templates  values.yaml

kubernetes-dashboard/charts/metrics-server/ci:
ci-values.yaml

kubernetes-dashboard/charts/metrics-server/templates:
NOTES.txt     apiservice.yaml                     clusterrole.yaml                        clusterrolebinding.yaml  pdb.yaml  rolebinding.yaml  serviceaccount.yaml
_helpers.tpl  clusterrole-aggregated-reader.yaml  clusterrolebinding-auth-delegator.yaml  deployment.yaml          psp.yaml  service.yaml

kubernetes-dashboard/templates:
NOTES.txt       clusterrole-metrics.yaml         clusterrolebinding-readonly.yaml  ingress.yaml        psp.yaml          secret.yaml          servicemonitor.yaml
_helpers.tpl    clusterrole-readonly.yaml        configmap.yaml                    networkpolicy.yaml  role.yaml         service.yaml
_tplvalues.tpl  clusterrolebinding-metrics.yaml  deployment.yaml                   pdb.yaml            rolebinding.yaml  serviceaccount.yaml
```

## Setting up a HULL Chart

Good, now proceed by creating a new empty HULL based Helm chart.  [The steps are documented here](https://github.com/vidispine/hull/blob/main/hull/doc/setup.md)
but you will create it from scratch here to understand what is needed.

One way to get started is by using the `helm create` command to create the folder structure for an empty Helm chart. Run this command to create our `kubernetes-dashboard-hull` Helm chart:

```sh
helm create kubernetes-dashboard-hull
```

should produce this answer:

```yml
Creating kubernetes-dashboard-hull
```

That should have created a folder `kubernetes-dashboard-hull` with the usual required and example Helm chart files in there. Switch to the folder and examine the contents:

```sh
cd kubernetes-dashboard-hull && ls -R
```

to find the default files for a new scaffolded Helm chart:

```yml
.:
Chart.yaml  charts  templates  values.yaml

./charts:

./templates:
NOTES.txt  _helpers.tpl  deployment.yaml  hpa.yaml  ingress.yaml  service.yaml  serviceaccount.yaml  tests

./templates/tests:
test-connection.yaml
```

Delete examples and tests in the `/templates` folder, nothing besides the `hull.yaml` is needed:

```sh
rm -rf templates/* && ls -R
```

shows this:

```yml
.:
Chart.yaml  charts  templates  values.yaml

./charts:

./templates:
```

Additionally empty the `values.yaml` with the example data:

```sh
echo '' > values.yaml
```

Now edit the mandatory `Chart.yaml` which is updated to reflect the original charts information. Furthermore add the HULL library as requirement which is necessary to be able to use HULL.

### Choosing a HULL version

When choosing a HULL version for this, it is recommended that HULLs `major.minor` version must at least match the `kubeVersion` field otherwise the API version of objects created by HULL may be yet unknown to the clusters API version. Each HULL release branch is aligned with a specific Kubernetes version such as `1.21`, `1.22`, `1.23` and so on. The objects that HULL renders out will always be of the latest API version of the given object type that matches the HULL librarys version. So when an API is upgraded from beta to stable in a Kubernetes version jump, the latter HULL version will create objects with the stable API version and the former will create the beta API versioned objects. It is recommended to leave the `kubeVersion` set at `1.21.0` for this example so you will be able to work against any Kubernetes cluster with a version of `1.21.0` or above.

Replace `1.23.3` below with the latest available HULL version (more on versioning in a later article) and match the `kubeVersion` accordingly:

```sh
echo 'apiVersion: v2
appVersion: 2.5.0
dependencies:
- condition: metrics-server.enabled
  name: metrics-server
  repository: https://kubernetes-sigs.github.io/metrics-server/
  version: 3.5.0
- name: hull
  version: "1.23.3"
  repository: "https://vidispine.github.io/hull"
description: General-purpose web UI for Kubernetes clusters
home: https://github.com/kubernetes/dashboard
icon: https://raw.githubusercontent.com/kubernetes/kubernetes/master/logo/logo.svg
keywords:
- kubernetes
- dashboard
kubeVersion: ">=1.21.0-0"
maintainers:
- email: cdesaintmartin@wiremind.fr
  name: desaintmartin
name: kubernetes-dashboard
sources:
- https://github.com/kubernetes/dashboard
version: 5.2.0' > Chart.yaml
```

Verify it was written correctly:
```sh
cat Chart.yaml
```

returns:

```yml
apiVersion: v2
appVersion: 2.5.0
dependencies:
- condition: metrics-server.enabled
  name: metrics-server
  repository: https://kubernetes-sigs.github.io/metrics-server/
  version: 3.5.0
- name: hull
  version: "1.23.3"
  repository: "https://vidispine.github.io/hull"
description: General-purpose web UI for Kubernetes clusters
home: https://github.com/kubernetes/dashboard
icon: https://raw.githubusercontent.com/kubernetes/kubernetes/master/logo/logo.svg
keywords:
- kubernetes
- dashboard
kubeVersion: ">=1.21.0-0"
maintainers:
- email: cdesaintmartin@wiremind.fr
  name: desaintmartin
name: kubernetes-dashboard
sources:
- https://github.com/kubernetes/dashboard
version: 5.2.0
```

Before starting work on the `values.yaml` you should download the HULL library by the Helm way of updating Chart dependencies:

```sh
helm dependency update
```

to download the HULL chart from the GitHub hosted HelmChart repository and the other subchart `metrics-server` which you can optionally co-deploy:

```yml
Getting updates for unmanaged Helm repositories...
...Successfully got an update from the "https://kubernetes-sigs.github.io/metrics-server/" chart repository
...Successfully got an update from the "https://vidispine.github.io/hull" chart repository
Saving 2 charts
Downloading metrics-server from repo https://kubernetes-sigs.github.io/metrics-server/
Downloading hull from repo https://vidispine.github.io/hull
Deleting outdated charts
```

Great, one tiny thing left before you can start defining the objects of our `kubernetes-dashboard-hull` Chart: copy over the `hull.yaml` from the now locally downloaded subchart to the main charts template folder. Without this step, nothing can be rendered via HULL since the `hull.yaml` contains the logic to convert `values.yaml` object definitions to rendered YAML files. We can do this with a one-liner where we extract the packaged HULL tgz file from the `/charts` directory to a local `temp` directory, copy over the `hull.yaml` and clean up the temp directory afterwards:

```sh
mkdir _tmp && tar -xvzf charts/hull-1.23.3.tgz -C _tmp && cp _tmp/hull/hull.yaml templates/hull.yaml && rm -rf _tmp
```

prints out the temporarily extracted files:

```yml
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
hull/templates/_objects.tpl
hull/templates/_objects_base_plain.tpl
hull/templates/_objects_base_pod.tpl
hull/templates/_objects_base_rbac.tpl
hull/templates/_objects_base_role.tpl
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
hull/CHANGELOG.md
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
hull/doc/objects_role.md
hull/doc/objects_service.md
hull/doc/objects_webhook.md
hull/doc/setup.md
hull/doc/transformations.md
hull/files/examples/authservice_values.yaml
hull/files/examples/medialogger_values.yaml
hull/files/scripts/HullChart.ps1
hull/files/scripts/SetChartVersion.ps1
hull/files/scripts/SetYamlValues.ps1
hull/files/templates/hull-clusterrole.yaml
hull/files/templates/hull-clusterrolebinding.yaml
hull/files/templates/hull-configmap.yaml
hull/files/templates/hull-cronjob.yaml
hull/files/templates/hull-customresource.yaml
hull/files/templates/hull-daemonset.yaml
hull/files/templates/hull-deployment.yaml
hull/files/templates/hull-endpoints.yaml
hull/files/templates/hull-horizontalpodautoscaler.yaml
hull/files/templates/hull-ingress.yaml
hull/files/templates/hull-ingressclass.yaml
hull/files/templates/hull-job.yaml
hull/files/templates/hull-mutatingwebhookconfiguration.yaml
hull/files/templates/hull-networkpolicy.yaml
hull/files/templates/hull-persistentvolume.yaml
hull/files/templates/hull-persistentvolumeclaim.yaml
hull/files/templates/hull-poddisruptionbudget.yaml
hull/files/templates/hull-podsecuritypolicy.yaml
hull/files/templates/hull-priorityclass.yaml
hull/files/templates/hull-registry.yaml
hull/files/templates/hull-resourcequota.yaml
hull/files/templates/hull-role.yaml
hull/files/templates/hull-rolebinding.yaml
hull/files/templates/hull-secret.yaml
hull/files/templates/hull-service.yaml
hull/files/templates/hull-serviceaccount.yaml
hull/files/templates/hull-servicemonitor.yaml
hull/files/templates/hull-statefulset.yaml
hull/files/templates/hull-storageclass.yaml
hull/files/templates/hull-validatingwebhookconfiguration.yaml
hull/hull.yaml
```

Now check if the `hull.yaml` was copied correctly to the `templates` folder:
```sh
cat templates/hull.yaml
```

should return:

```yml
{{- include "hull.objects.prepare.all" (dict "HULL_ROOT_KEY" "hull" "ROOT_CONTEXT" $) }}
```

### Inspecting default RBAC settings

Fantastic, all in place! If everything went right up to here you can take a first look at what HULL does by default in an otherwise unconfigured Helm chart. For this you can have Helm render out the YAMLs to `stdout` where you can study them instead of directly deploying objects to a cluster. `helm template` is the command used for this rendering out. If you have an error complaining about version incompatibility you have to update your Helm installation so that `helm template` runs against a new API version.

Go on by templating out what we have in our "empty" chart so far:

```sh
helm template .
```

giving us an unexpected amount of objects:

```yml
---
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: release-name-metrics-server
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/clusterrole-aggregated-reader.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server-aggregated-reader
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
  - apiGroups:
      - metrics.k8s.io
    resources:
      - pods
      - nodes
    verbs:
      - get
      - list
      - watch
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:release-name-metrics-server
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
rules:
  - apiGroups:
    - ""
    resources:
      - pods
      - nodes
      - nodes/stats
      - namespaces
      - configmaps
    verbs:
      - get
      - list
      - watch
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/clusterrolebinding-auth-delegator.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: release-name-metrics-server:system:auth-delegator
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: release-name-metrics-server
    namespace: default
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:release-name-metrics-server
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:release-name-metrics-server
subjects:
  - kind: ServiceAccount
    name: release-name-metrics-server
    namespace: default
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
rules: []
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: release-name-metrics-server-auth-reader
  namespace: kube-system
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
  - kind: ServiceAccount
    name: release-name-metrics-server
    namespace: default
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: release-name-kubernetes-dashboard-default
subjects:
- kind: ServiceAccount
  name: release-name-kubernetes-dashboard-default
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: release-name-metrics-server
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
spec:
  type: ClusterIP
  ports:
    - name: https
      port: 443
      protocol: TCP
      targetPort: https
  selector:
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: release-name-metrics-server
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: metrics-server
      app.kubernetes.io/instance: release-name
  template:
    metadata:
      labels:
        app.kubernetes.io/name: metrics-server
        app.kubernetes.io/instance: release-name
    spec:
      serviceAccountName: release-name-metrics-server
      priorityClassName: "system-cluster-critical"
      containers:
        - name: metrics-server
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
          image: k8s.gcr.io/metrics-server/metrics-server:v0.5.0
          imagePullPolicy: IfNotPresent
          args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --kubelet-use-node-status-port
            - --metric-resolution=15s
          ports:
          - name: https
            protocol: TCP
            containerPort: 4443
          livenessProbe:
            failureThreshold: 3
            httpGet:
              path: /livez
              port: https
              scheme: HTTPS
            initialDelaySeconds: 0
            periodSeconds: 10
          readinessProbe:
            failureThreshold: 3
            httpGet:
              path: /readyz
              port: https
              scheme: HTTPS
            initialDelaySeconds: 20
            periodSeconds: 10
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
---
# Source: kubernetes-dashboard/charts/metrics-server/templates/apiservice.yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
  labels:
    helm.sh/chart: metrics-server-3.5.0
    app.kubernetes.io/name: metrics-server
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/version: "0.5.0"
    app.kubernetes.io/managed-by: Helm
spec:
  group: metrics.k8s.io
  groupPriorityMinimum: 100
  insecureSkipTLSVerify: true
  service:
    name: release-name-metrics-server
    namespace: default
  version: v1beta1
  versionPriority: 100
```

Quite a lot of objects, this is because the templating command rendered both the HULL default RBAC objects as well as the `metrics-server` subchart objects. Since we did not explicitly enable the deployment of the `metrics-server` subchart this can be considered a bug, however let us just disable the `metrics-server` chart deployment for this tutorial by typing:

```sh
echo 'metrics-server:
  enabled: false' > values.yaml
```

Now the `hull template` output should only contain the HULL objects which are created by default. You can verify this by entering again:

```sh
helm template .
```

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
rules: []
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: release-name-kubernetes-dashboard-default
subjects:
- kind: ServiceAccount
  name: release-name-kubernetes-dashboard-default
```

There is already a lot you can see from this output. One thing you may ask is: why are there already objects created while I haven't even defined anything yet? That's because by default, HULL will render a 'default' set of RBAC objects which you can use and fine-tune for your application. The objects that are automatically created are:
- a default ServiceAccount with name `<release-name>-<chart-name>-default` which is preconfigured for all pods as the `serviceAccountName` to use
- a default Role with name `<release-name>-<chart-name>-default` which initially has no rules but you can add them as needed
- a default RoleBinding with name `<release-name>-<chart-name>-default` that connects the `<release-name>-<chart-name>-default` Role with the `<release-name>-<chart-name>-default` ServiceAccount

This is like a RBAC bootstrap you can build upon easily - but you may have different needs so if this setup does not fit your usecase, you can disable this default set of RBAC objects and specify different RBAC objects more tailored to your needs. The investigation of options for RBAC configuration are the subject of an upcoming tutorial in this series.

However, while using RBAC is the default with HULL you can also optionally turn off the rendering of RBAC objects all together if that is what you want (of course not recommended :) ). HULL offers the following chart global switch to opt out of RBAC and if `false` RBAC related objects will not be rendered anymore:

```yml
hull:
  config:
    general:
      rbac: false
```

### Inspecting the autogenerated metadata

Another standout aspect here is the autogenerated metadata for all objects which adheres to the [best practice Kubernetes metadata definition](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/) and [Helm's recommendation on the subject matter](https://helm.sh/docs/chart_best_practices/labels/#standard-labels). Using HULL you get this default metadata created without you having to do anything and on top of this it is easy to [enrich all objects or objects of a particular type or just individual objects with metadata even further](https://github.com/vidispine/hull/tree/main/hull#advanced-example) which is also covered in the remainder of this tutorial. 

For example, check the `metadata` block of the ServiceAccount that was rendered:

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: default
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-default
```

Autogenerating standard metadata and being able to define inheritable metadata is a convenience feature that enforces uniformity of objects and thus operations on them.

## Wrap up

That's it for Part 2, Part 3 looks at ConfigMaps and Secrets and you will add your first custom objects to the `kubernetes-dashboard-hull` Helm chart! You can check out the complete code created so far as the outcome of this Part 2 [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/02_setup).

Thanks a lot for reading!