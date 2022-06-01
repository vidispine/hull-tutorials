## Preparation

After the ConfigMaps and Secrets tutorial part this part will look at the RBAC configuration side of `kubernetes-dashboard`'s Helm chart and check how it is transportable to the new HULL based `kubernetes-dashboard-hull` chart.

Initially you should copy over our working chart from the last tutorial part to a new folder in the same manner as we did before. Go to your working directory and execute:

```sh
cd ~/kubernetes-dashboard-hull && cp -R 03_configmaps_and_secrets/ 04_rbac && cd 04_rbac/kubernetes-dashboard-hull
```

From the `values.full.yaml`, the starting point for this tutorial part, only keep the `hull.config.specific` section for building upon in this chapter, the `hull.objects` section should be cleared for now since you will only add objects relevant to this tutorial section to it. At the end of this tutorial part the newly created objects and newly added `hull.config.specific` properties are added to the `values.full.yaml` with what was already created. This in turn will then serve as the next tutorial parts starting point.

To delete the HULL objects from `values.full.yaml` and copy the rest of our `values.yaml` to the `values.yaml` type the following:

```sh
sed '/  objects:/Q' values.full.yaml > values.yaml
```

and verify the result in the new `values.yaml`:

```sh
cat values.yaml
```

to show only the `hull.config.specific` section:

```yml
metrics-server:
  enabled: false
hull:
  config:
    specific:
      settings: {}
      pinnedCRDs: {}
```

Let's get started on ServiceAccount configuration and role based access control!

## Examining the existing RBAC objects

Execute the following command to first get an idea which RBAC objects are defined:

```sh
find ../kubernetes-dashboard/templates -type f -iregex '.*\(Role\|\role|\Account\|account\|rbac\|RBAC\).*' | sort
```

which reveals them all:

```yml
../kubernetes-dashboard/templates/clusterrole-metrics.yaml
../kubernetes-dashboard/templates/clusterrole-readonly.yaml
../kubernetes-dashboard/templates/clusterrolebinding-metrics.yaml
../kubernetes-dashboard/templates/clusterrolebinding-readonly.yaml
../kubernetes-dashboard/templates/role.yaml
../kubernetes-dashboard/templates/rolebinding.yaml
../kubernetes-dashboard/templates/serviceaccount.yaml
```

In the previous tutorial part when you created the `kubernetes-dashboard-hull` chart you already had a glance at the default RBAC configuration that comes with HULL. So now is the time to adapt and tweak it so it matches the original Helm charts RBAC functionally as closely as possible.

First check the ServiceAccount:

```sh
cat ../kubernetes-dashboard/templates/serviceaccount.yaml
```

which is this:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
  name: {{ template "kubernetes-dashboard.serviceAccountName" . }}
{{- end -}}
```

The actual creation of the ServiceAccount name is deferred to a helper function in `_helpers.tpl` which we look at in more detail:

```sh
cat ../kubernetes-dashboard/templates/_helpers.tpl | grep 'kubernetes-dashboard\.serviceAccountName' -B 4 -A 10
```

where the inherent logic becomes more apparent:

```yml

{{/*
Name of the service account to use
*/}}
{{- define "kubernetes-dashboard.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
    {{ default (include "kubernetes-dashboard.fullname" .) .Values.serviceAccount.name }}
{{- else -}}
    {{ default "default" .Values.serviceAccount.name }}
{{- end -}}
{{- end -}}
```

Now check the `values.yaml` section dealing with the `serviceAccount` configuration: 

```sh
cat ../kubernetes-dashboard/values.yaml | grep '^serviceAccount' -B 1 -A 6
```

for completing our examination of the `controller`s ServiceAccount configuration:

```yml

serviceAccount:
  # Specifies whether a service account should be created
  create: true
  # The name of the service account to use.
  # If not set and create is true, a name is generated using the fullname template
  name:

```

Now you should have garnered an impression of the ServiceAccount configuration options and relations which is actually a typical way to model this aspect in Helm charts. Let's look at how you can handle the ServiceAccount and RBAC aspects with HULL.

A word in advance: while it is useful to omit rendering of the `default` ServiceAccount, Role and RoleBinding objects for the most part of this tutorial it does not make sense to do this when checking the RBAC configuration options that do involve them. Therefore during this turorial section the `-f ../configs/disable-default-rbac.yaml` parameter is not being set and thus the `default` RBAC objects and ServiceAccount are rendered. For the next tutorial part add the switch again to reduce output.

## Choosing ServiceAccounts for pods in HULL

When breaking down the logic within function `kubernetes-dashboard.serviceAccountName` and the `serviceaccount.yaml`, the following four variants of handling ServiceAccount configuration and naming can be deduced (in _cursive_ the conditions that apply to each variant): 

1. create a new individual ServiceAccount for this chart named as `<release-name>-<chart-name>`.

    _Settingwise this is done by enabling `serviceAccount.create`  and not providing a `serviceAccount.name`)._ 

    > This is the default in `kubernetes-dashboard`'s `values.yaml`.

2. create a new individual ServiceAccount for this chart with a fixed provided name. 

    _Achieved by setting `serviceAccount.create` to `true` and providing a `serviceAccount.name`._

3. use a given ServiceAccount with a fixed provided name. 

    _For this choice set `serviceAccount.create` to `false` and provide the full `serviceAccount.name` to use._

4. use the Kubernetes default ServiceAccount simply named 'default'. 

    _Setting `serviceAccount.create` to `false` and not supplying a `serviceAccount.name` has this result._

Now within HULL the ServiceAccount for a pod is derived generally by the following rules:

1. if no specific `serviceAccountName` property is set on the pod it is checked whether `hull.objects.serviceaccount.default.enabled` is `true` (defaults to `true`) and if so the chart and release specific default ServiceAccount is being used (which is named `<release-name>-<chart-name>-default`). A matching `default` Role and RoleBinding is also always created by default as was shown already.

   > This is the default in HULL's `values.yaml`

2. if a specific `serviceAccountName` property is set on the pod it is used for that pod. Note that this field requires the exact full name to be set. So in case you want to create and use a ServiceAccount which is dynamically created within the same chart you need to use the fullname transformation (short form `_HT^`) from HULL to generate this name in place. For example to configure a ServiceAccount `myuser` for a pod use `serviceAccountName: _HT^myuser` which would by default produce `serviceAccountName: <release-name>-<chart-name>-myuser` in the rendered output. The fullname transformation `_HT^` was introduced in the last tutorial section.

3. lastly if the release specific default ServiceAccount is disabled, the property `serviceAccountName` is not set which will default in Kubernetes to using the always existing Kubernetes standard ServiceAccount simply named 'default' - don't mistake it with the chart specific `default` ServiceAccount HULL creates!

### Using the release specific default ServiceAccount

Comparing the logic between the `kubernetes-dashboard` and HULL based ServiceAccount handling you can first see that the default with HULL is very similar to the default in the original chart (case 1.). The only minor difference here is that the `-default` suffix is added to the newly created ServiceAccount by HULL.

Using a test overlay file named `../configs/default-sa.yaml` you can produce this default case. For now a roughly sketched `dashboard` Deployment with a basic pod spec serves to allow setting the `serviceAccountName` property on an actual pod.

> Workloads such as the `dashboard` Deployment will be part of a later tutorial so you can only concentrate on the `serviceAccountName` property handling for now.

Insert a first simple deployment with a pod's ServiceAccount to visualize the different variants:

```sh
echo 'hull:
  objects:
    deployment:
      dashboard:
        pod:
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: "v2.5.0"' > ../configs/default-sa.yaml \
&& helm template -f ../configs/default-sa.yaml .
```

and you'll see the the charts `default` ServiceAccount being used for the pod (actually all pods if there were more and they were setup in the same way):

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
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-dashboard
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: dashboard
        app.kubernetes.io/instance: release-name
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: kubernetes-dashboard
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 2.5.0
        helm.sh/chart: kubernetes-dashboard-5.2.0
    spec:
      containers:
      - env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        name: dashboard
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: release-name-kubernetes-dashboard-default
      volumes: []
```

So `serviceAccountName: release-name-kubernetes-dashboard-default` is used for the pod by default which is as intended. The `default` Role and RoleBindings are auto-created and available for more detailed RBAC configuration of the `default` ServiceAccount in case you choose to use RBAC. 

The other original cases 2. - 4. can be realized with HULL too by reconfiguring the chart at deploy time. How to do so will be examined next.

### Using the Kubernetes 'default' ServiceAccount

When setting `hull.objects.serviceaccount.default.enabled` to `false` and not setting any `serviceAccountName` on pods, the Kubernetes standard ('default') account is used for all pods (matching original case 4.). If you don't want to have or use the `default` ServiceAccount at all in your applications Helm Chart but still use RBAC it makes sense to disable the `default` Role and RoleBindings as well so they do not point at a nonexisting ServiceAccount. Here we explicitly disable them in the upcoming examples if we don't need them for the particular example so there will be less output to interpret. Also, if you don't use RBAC at all, disabling the `hull.config.specific.rbac` setting will also prevent rendering of all (Cluster)Roles and (Cluster)RoleBindings.

Create a pod using the standard Kubernetes 'default' ServiceAccount like this:

```sh
echo 'hull:
  objects:
    serviceaccount: 
      default:
        enabled: false
    role: 
      default:
        enabled: false
    rolebinding: 
      default:
        enabled: false
    deployment:
      dashboard:
        pod:
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: "v2.5.0"' > ../configs/k8s-default-sa.yaml \
&& helm template -f ../configs/k8s-default-sa.yaml .
```

and you'll find that no `default` ServiceAccount, Role or RoleBinding was created and no ServiceAccount is set on the pod so the 'default' one from Kubernetes is applied as intended:

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-dashboard
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: dashboard
        app.kubernetes.io/instance: release-name
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: kubernetes-dashboard
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 2.5.0
        helm.sh/chart: kubernetes-dashboard-5.2.0
    spec:
      containers:
      - env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        name: dashboard
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      volumes: []
```

### Using a ServiceAccount defined outside of Helm Chart

To produce the original case 3 where an externally defined ServiceAccount is put to use, setting explicit `serviceAccountName`'s on individual pods does the trick. It is up to you if you want to set the `default` ServiceAccount, Role or RoleBinding to disabled but let's do it here since we don't plan on using them elsewhere in this example and we can reduce `helm template` output:

```sh
echo 'hull:
  objects:
    serviceaccount: 
      default:
        enabled: false
    role: 
      default:
        enabled: false
    rolebinding: 
      default:
        enabled: false
    deployment:
      dashboard:
        pod:
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: "v2.5.0"
          serviceAccountName: my-external-cluster-sa' > ../configs/external-sa.yaml \
&& helm template -f ../configs/external-sa.yaml .
```

yielding the result expected:

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-dashboard
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: dashboard
        app.kubernetes.io/instance: release-name
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: kubernetes-dashboard
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 2.5.0
        helm.sh/chart: kubernetes-dashboard-5.2.0
    spec:
      containers:
      - env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        name: dashboard
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: my-external-cluster-sa
      volumes: []
```

The static name for the ServiceAccount was used. If you manage ServiceAccounts outside of the particular Helm chart this is a typical scenario.

### Using release specific non-default ServiceAccounts

Last case to cover is case 2. where you may want to create one or more ServiceAccounts with the Helm chart release but intend to not use the `default` ServiceAccount anywhere. To emulate this, setting `hull.objects.serviceaccount.default.enabled` to `false` and creating one or more ServiceAccounts for RBAC assignment with the desired names is required.

In terms of choosing the name for a ServiceAccount to be created and used in the pod spec there are two choices which are both covered by HULL. You may opt for creating ServiceAccount(s) with unique names that are release associated by dynamic naming or for creating ServiceAccount(s) with the exact names you want.

#### Specifying unique and non-default dynamic ServiceAccounts

To create a dynamic name (with pattern `<release-name>-<chart-name>-<component-name>`) it is required to apply a transformation on the `serviceAccountName` on the individual pods to include the dynamic prefix in the rendered result like in this example:

```sh
echo 'hull:
  objects:
    serviceaccount:
      default:
        enabled: false
      other_sa:
        automountServiceAccountToken: true
    role:
      default:
        enabled: false
    rolebinding:
      default:
        enabled: false
    deployment:
      dashboard:
        pod:
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: "v2.5.0"
          serviceAccountName: _HT^other_sa' > ../configs/internal-dynamic-sa.yaml \
&& helm template -f ../configs/internal-dynamic-sa.yaml .
```

`_HT^` is used to create the release specific name with a given suffix (here `other_sa`):

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: other_sa
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-other_sa
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-dashboard
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: dashboard
        app.kubernetes.io/instance: release-name
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: kubernetes-dashboard
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 2.5.0
        helm.sh/chart: kubernetes-dashboard-5.2.0
    spec:
      containers:
      - env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        name: dashboard
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: release-name-kubernetes-dashboard-other_sa
      volumes: []
```

As you can see the name of the created ServiceAccount `name: release-name-kubernetes-dashboard-hull-other_sa` matches the pods `serviceAccountName: release-name-kubernetes-dashboard-hull-other_sa` exactly as intended. In this scenario you may want to proceed defining Role and RoleBindings as required for the `other_sa` ServiceAccount.

#### Specifying non-default static ServiceAccounts

The other alternative is to create a completely static ServiceAccount name (without the `<release-name>-<chart-name>-` prefix) and use this for the pod(s). For this you do not need the `makefullname`/`_HT^` transformation of the pods `serviceAccountName` to use the name as is but you additionally need add the `staticName: true` property to the ServiceAccount objects definition as in this example:

```sh
echo 'hull:
  objects:
    serviceaccount:
      default:
        enabled: false
      other_sa:
        staticName: true
        automountServiceAccountToken: true
    role:
      default:
        enabled: false
    rolebinding:
      default:
        enabled: false
    deployment:
      dashboard:
        pod:
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: "v2.5.0"
          serviceAccountName: other_sa' > ../configs/internal-static-sa.yaml \
&& helm template -f ../configs/internal-static-sa.yaml .
```

and find the statically named ServiceAccount configured for the pod:

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: other_sa
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: other_sa
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-dashboard
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: dashboard
        app.kubernetes.io/instance: release-name
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: kubernetes-dashboard
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 2.5.0
        helm.sh/chart: kubernetes-dashboard-5.2.0
    spec:
      containers:
      - env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        name: dashboard
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: other_sa
      volumes: []
```

> The caveat to this approach is that you need to choose carefully which ServiceAccount name and object to create because it may already exist in the namespace as opposed to sticking with dynamic names only where object names are highly unique. If you choose a static name that already exists in the namespace that will cause problems at deploy time!

Except for the default case of creating a dynamic and unique default ServiceAccount, the other ServiceAccount handling cases require little additional configuration. However, the HULL based configuration is transparent, explicit and flexible if special needs exist and the full flexibility of ServiceAccount management is needed to adapt to a given scenario. Otherwise providing pre-defined RBAC objects for the `default` ServiceAccount and using this ServiceAccount in the pods is likely sufficient configuration for smaller application deployments.

## Defining a RoleBinding

Next up is adding more RBAC objects to the mix in form of the RoleBinding:

```sh
cat ../kubernetes-dashboard/templates/rolebinding.yaml
```

which is:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ template "kubernetes-dashboard.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ template "kubernetes-dashboard.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end -}}
```

In HULL RBAC is globally enabled by a `true`/`false` switch which is built into HULL as a predefined key at `hull.config.general.rbac`. If set to `true`, the RBAC related objects are rendered, if set to false they are just omitted from rendering. This mimics the behaviour that is very common in many popular Helm charts and is implemented by the conditional rendering switch here:

```yml
{{ if .Values.rbac.create -}}
```

> The recommendation is to always leave RBAC enabled in your clusters.

From the above `cat` output you can see that the RoleBinding and referenced Role name are again dependent on the `kubernetes-dashboard.fullname` function and the ServiceAccount name is subject to the `kubernetes-dashboard.serviceAccountName` function examined earlier. Unless you need to or want to change the controllers ServiceAccount name from the default there is nothing to do here except having RBAC enabled to create a sufficient `default` RoleBinding in HULL looking like this:

```yml
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

If you want to deviate from the default handling you can freely overwrite the `roleRef` and `subjects` as needed, the same techniques explained above do apply for the creation of object names.

## Configuring Roles

Now for the Role:

```sh
cat ../kubernetes-dashboard/templates/role.yaml
```

which returns:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
rules:
    # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["get", "update"]
    # Allow Dashboard to get metrics.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster", "dashboard-metrics-scraper"]
    verbs: ["proxy"]
  - apiGroups: [""]
    resources: ["services/proxy"]
    resourceNames: ["heapster", "http:heapster:", "https:heapster:", "dashboard-metrics-scraper", "http:dashboard-metrics-scraper"]
    verbs: ["get"]
{{- end -}}
```

You can now use the `default` Role that comes with HULL in the standard scenario and add the `rules:` as shown here and look at interesting aspects afterwards:

```sh
echo '  objects:
    role:
      default:
        rules:
          secrets:
            apiGroups:
            - ""
            resources:
            - secrets
            resourceNames: _HT![ 
              {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "certs") }},
              "kubernetes-dashboard-csrf",
              "kubernetes-dashboard-key-holder"
              ]
            verbs:
            - get
            - update
            - delete
          configmaps:
            apiGroups:
            - ""
            resources:
            - configmaps
            resourceNames: _HT![
              {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "settings") }} ]
            verbs:
            - get
            - update
          services:
            apiGroups:
            - ""
            resources:
            - services
            resourceNames:
            - "heapster"
            - "dashboard-metrics-scraper"
            verbs:
            - proxy
          services_proxy:
            apiGroups:
            - ""
            resources:
            - services/proxy
            resourceNames:
            - "heapster"
            - "http:heapster:"
            - "https:heapster:"
            - "dashboard-metrics-scraper"
            - "http:dashboard-metrics-scraper"
            verbs:
            - get' >> values.yaml
```

The `services` and `services_proxy` part of this configuration is static mappings to services which are defined in the `metrics-server` subchart, no need to focus on them now. The usage of the `_HT!` transformations is interesting though: utilizing the full power of the `tpl` function an array is dynamically created in the compact flow style `[]` notation and the elements are individually processed. If you remember the part where the Secrets where created there was one with a dynamic name (`certs`) and two where a static name was demanded (`csrf` and `key-holder`). Therefore you can see in the above specification that the dynamic names are created using the `hull.metadata.fullname` function being called within the `_HT!` `tpl` transformation.

The `hull.metadata.fullname` is also what is called internally when using the HULL transformation `hull.util.transformation.makefullname`/`_HT^`. Here you need to use it directly within the `tpl` function which is however also straightforward since it has a fixed signature of:

```yml
include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "<COMPONENT>")
```

and you only need to set the `<COMPONENT>` part to the objects key you are referring to.

As a side note on the original template: be aware that the fact that the `certs` Secrets name is refered to as `kubernetes-dashboard-certs` - a static expression - with:

```yml
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder", "kubernetes-dashboard-certs", "kubernetes-dashboard-csrf"]
```

is problematic. Looking back at the `name` specification of the `cert` Secret: 

```yml
  name: {{ template "kubernetes-dashboard.fullname" . }}-certs
```

there is actually no guarantee that the Secrets name is constructed like this. Depending on overall chart configuration (`fullnameOverride`) the static reference to a dynamic name may be invalid and not point to the correct name of the `certs` Secret. So this is a "bug" in the sense that it can lead to a potentially unusable release. Due to the huge amount of manual template creation in the regular Helm workflow problems such as these arise often where different parts of the templates with overlapping concerns are not properly matched or updated together.

> Note that with the HULL object and reference handling this would not happen here since name generation (the object instance name and the reference name) is handled via the same `hull.metadata.fullname` function so the reference is always a stable one.

Finally if you print the output of above:

```sh
helm template .
```

and review whether the desired outcome was rendered:

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
rules:
- apiGroups:
  - ""
  resourceNames:
  - release-name-kubernetes-dashboard-settings
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resourceNames:
  - release-name-kubernetes-dashboard-certs
  - kubernetes-dashboard-csrf
  - kubernetes-dashboard-key-holder
  resources:
  - secrets
  verbs:
  - get
  - update
  - delete
- apiGroups:
  - ""
  resourceNames:
  - heapster
  - dashboard-metrics-scraper
  resources:
  - services
  verbs:
  - proxy
- apiGroups:
  - ""
  resourceNames:
  - heapster
  - 'http:heapster:'
  - 'https:heapster:'
  - dashboard-metrics-scraper
  - http:dashboard-metrics-scraper
  resources:
  - services/proxy
  verbs:
  - get
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

everything should be as intended. 

On to the last aspect of this tutorial part, the cluster-level RBAC objects!

## Writing ClusterRole and ClusterRoleBindings

Structurally ClusterRole and ClusterRoleBinding objects are nearly identical to their "non-prefixed with Cluster" counterparts. 

### ClusterRoles
For the [ClusterRole/Role difference the Kubernetes documentation](https://v1-18.docs.kubernetes.io/docs/reference/access-authn-authz/rbac/) states:

> A Role always sets permissions within a particular namespace; when you create a Role, you have to specify the namespace it belongs in.

ClusterRole, by contrast, is a non-namespaced resource. The resources have different names (Role and ClusterRole) because a Kubernetes object always has to be either namespaced or not namespaced; it can't be both.

Furthermore ClusterRole has an additional property `aggregationRule` which the Role object does not have.

### ClusterRoleBinding

For the ClusterRoleBinding the `roleRef` needs to reference a global ClusterRole and not a Role.

### Converting the existing Cluster-scope RBAC objects

Recaping the Cluster based RBAC objects:

```sh
find ../kubernetes-dashboard/templates -type f -iregex '.*\(ClusterRole\|\clusterrole\).*' | sort
```

you can see they fall into two "groups", the `metrics` and `readonly` objects:

```yml
../kubernetes-dashboard/templates/clusterrole-metrics.yaml
../kubernetes-dashboard/templates/clusterrole-readonly.yaml
../kubernetes-dashboard/templates/clusterrolebinding-metrics.yaml
../kubernetes-dashboard/templates/clusterrolebinding-readonly.yaml
```

First convert the `readonly` objects from the `/templates` folder, starting with the ClusterRole:

```sh
cat ../kubernetes-dashboard/templates/clusterrole-readonly.yaml
```

showing:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.rbac.clusterReadOnlyRole -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: "{{ template "kubernetes-dashboard.fullname" . }}-readonly"
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - persistentvolumeclaims
      - pods
      - replicationcontrollers
      - replicationcontrollers/scale
      - serviceaccounts
      - services
      - nodes
      - persistentvolumeclaims
      - persistentvolumes
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - bindings
      - events
      - limitranges
      - namespaces/status
      - pods/log
      - pods/status
      - replicationcontrollers/status
      - resourcequotas
      - resourcequotas/status
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - namespaces
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - apps
    resources:
      - daemonsets
      - deployments
      - deployments/scale
      - replicasets
      - replicasets/scale
      - statefulsets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - autoscaling
    resources:
      - horizontalpodautoscalers
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - batch
    resources:
      - cronjobs
      - jobs
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - extensions
    resources:
      - daemonsets
      - deployments
      - deployments/scale
      - ingresses
      - networkpolicies
      - replicasets
      - replicasets/scale
      - replicationcontrollers/scale
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - policy
    resources:
      - poddisruptionbudgets
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - networkpolicies
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - storage.k8s.io
    resources:
      - storageclasses
      - volumeattachments
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - rbac.authorization.k8s.io
    resources:
      - clusterrolebindings
      - clusterroles
      - roles
      - rolebindings
    verbs:
      - get
      - list
      - watch
{{- end -}}
```

The whole ClusterRole is only rendered given that `.Values.rbac.clusterReadOnlyRole` is true, looking at it:

```sh
cat ../kubernetes-dashboard/values.yaml | grep "clusterReadOnlyRole" -B 13 -A 1
```

it comes with following description:

```yml

rbac:
  # Specifies whether namespaced RBAC resources (Role, Rolebinding) should be created
  create: true

  # Specifies whether cluster-wide RBAC resources (ClusterRole, ClusterRolebinding) to access metrics should be created
  # Independent from rbac.create parameter.
  clusterRoleMetrics: true

  # Start in ReadOnly mode.
  # Specifies whether cluster-wide RBAC resources (ClusterRole, ClusterRolebinding) with read only permissions to all resources listed inside the cluster should be created
  # Only dashboard-related Secrets and ConfigMaps will still be available for writing.
  #
  # The basic idea of the clusterReadOnlyRole
  # is not to hide all the secrets and sensitive data but more
  # to avoid accidental changes in the cluster outside the standard CI/CD.
  #
  # It is NOT RECOMMENDED to use this version in production.
  # Instead you should review the role and remove all potentially sensitive parts such as
  # access to persistentvolumes, pods/log etc.
  #
  # Independent from rbac.create parameter.
  clusterReadOnlyRole: false
```

As expected, when checking the ClusterRoleBinding:

```sh
cat ../kubernetes-dashboard/templates/clusterrolebinding-readonly.yaml
```

the same `{{ if .Values.rbac.clusterReadOnlyRole -}}` condition applies:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.rbac.clusterReadOnlyRole -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}-readonly
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ template "kubernetes-dashboard.fullname" . }}-readonly
subjects:
  - kind: ServiceAccount
    name: {{ template "kubernetes-dashboard.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
```

Thus it makes much sense to model the `rbac.clusterReadOnlyRole` switch in our `hull.config.specific` section so we can centrally disable and enable it. Otherwise nothing new or unexpected happens here so it is straightforward to convert this to a HULL based chart logic but before that check on the `metrics` components.

Inspecting the `metrics` components you see that they behave the same way: rendering is controlled by a global switch `{{ if .Values.rbac.clusterRoleMetrics -}}` which is also contained in the last `cat` of `values.yaml` and the rest is similar mapping of rules to the ClusterRole and ClusterRoleBinding between the ClusterRole and the ServiceAccount:

```sh
cat ../kubernetes-dashboard/templates/clusterrole-metrics.yaml
```

gives:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.rbac.clusterRoleMetrics -}}
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: "{{ template "kubernetes-dashboard.fullname" . }}-metrics"
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
rules:
  # Allow Metrics Scraper to get metrics from the Metrics server
  - apiGroups: ["metrics.k8s.io"]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
{{- end }}
```

and:

```sh
cat ../kubernetes-dashboard/templates/clusterrolebinding-metrics.yaml
```

gives:

```yml
# Copyright 2017 The Kubernetes Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

{{ if .Values.rbac.clusterRoleMetrics -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: "{{ template "kubernetes-dashboard.fullname" . }}-metrics"
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ template "kubernetes-dashboard.fullname" . }}-metrics
subjects:
  - kind: ServiceAccount
    name: {{ template "kubernetes-dashboard.serviceAccountName" . }}
    namespace: {{ .Release.Namespace }}
{{- end }}
```
In terms of converting this, first add in the new "global/specific" `rbac.clusterReadOnlyRole` and `rbac.clusterRoleMetrics` parameters to our `hull.config.specific` section:

```sh
sed '/^\s\s\s\sspecific/r'<(
      echo "      rbac:"
      echo "        clusterReadOnlyRole: false"
      echo "        clusterRoleMetrics: true"
    ) -i -- values.yaml
```

and then add the `readonly` ClusterRole and ClusterRoleBinding with the enabled switch bound to the `clusterReadOnlyRole` condition and the `metrics` ClusterRole and ClusterRoleBinding with the enabled switch bound to the `clusterRoleMetrics` condition:

```sh
echo '    clusterrole:
      readonly:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterReadOnlyRole
        rules:
          cluster_objects:
            apiGroups:
            - ""
            resources:
            - configmaps
            - endpoints
            - persistentvolumeclaims
            - pods
            - replicationcontrollers
            - replicationcontrollers/scale
            - serviceaccounts
            - services
            - nodes
            - persistentvolumeclaims
            - persistentvolumes
            verbs:
            - get
            - list
            - watch
          statuses:
            apiGroups:
            - ""
            resources:
            - bindings
            - events
            - limitranges
            - namespaces/status
            - pods/log
            - pods/status
            - replicationcontrollers/status
            - resourcequotas
            - resourcequotas/status
            verbs:
            - get
            - list
            - watch
          namespaces:
            apiGroups:
            - ""
            resources:
            - namespaces
            verbs:
            - get
            - list
            - watch
          apps:
            apiGroups:
            - apps
            resources:
            - daemonsets
            - deployments
            - deployments/scale
            - replicasets
            - replicasets/scale
            - statefulsets
            verbs:
            - get
            - list
            - watch
          autoscaling:
            apiGroups:
            - autoscaling
            resources:
            - horizontalpodautoscalers
            verbs:
            - get
            - list
            - watch
          jobs:
            apiGroups:
            - batch
            resources:
            - cronjobs
            - jobs
            verbs:
            - get
            - list
            - watch
          extensions:
            apiGroups:
            - extensions
            resources:
            - daemonsets
            - deployments
            - deployments/scale
            - ingresses
            - networkpolicies
            - replicasets
            - replicasets/scale
            - replicationcontrollers/scale
            verbs:
            - get
            - list
            - watch
          pdb:
            apiGroups:
            - policy
            resources:
            - poddisruptionbudgets
            verbs:
            - get
            - list
            - watch
          network:
            apiGroups:
            - networking.k8s.io
            resources:
            - networkpolicies
            - ingresses
            verbs:
            - get
            - list
            - watch
          storage:
            apiGroups:
            - storage.k8s.io
            resources:
            - storageclasses
            - volumeattachments
            verbs:
            - get
            - list
            - watch
          rbac:
            apiGroups:
            - rbac.authorization.k8s.io
            resources:
            - clusterrolebindings
            - clusterroles
            - roles
            - rolebindings
            verbs:
            - get
            - list
            - watch
      metrics:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterRoleMetrics
        rules:
          metrics:
            apiGroups:
            - "metrics.k8s.io"
            resources:
            - pods
            - nodes
            verbs:
            - get
            - list
            - watch
    clusterrolebinding:
      readonly:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterReadOnlyRole
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^readonly
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^default
      metrics:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterRoleMetrics
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^metrics
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^default' >> values.yaml 
```

If you want to check the new objects out, you can enable them both:

```sh
echo 'hull:
  config:
    specific:
      rbac:
        clusterReadOnlyRole: true' > ../configs/enable-cluster-role.yaml \
&& helm template -f ../configs/enable-cluster-role.yaml .
```

and examine it:

```yaml
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
kind: ClusterRole
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: metrics
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-metrics
  namespace: ""
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
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: readonly
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-readonly
  namespace: ""
rules:
- apiGroups:
  - apps
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - replicasets
  - replicasets/scale
  - statefulsets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - autoscaling
  resources:
  - horizontalpodautoscalers
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - configmaps
  - endpoints
  - persistentvolumeclaims
  - pods
  - replicationcontrollers
  - replicationcontrollers/scale
  - serviceaccounts
  - services
  - nodes
  - persistentvolumeclaims
  - persistentvolumes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources:
  - daemonsets
  - deployments
  - deployments/scale
  - ingresses
  - networkpolicies
  - replicasets
  - replicasets/scale
  - replicationcontrollers/scale
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - batch
  resources:
  - cronjobs
  - jobs
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - namespaces
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - networkpolicies
  - ingresses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - policy
  resources:
  - poddisruptionbudgets
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - rbac.authorization.k8s.io
  resources:
  - clusterrolebindings
  - clusterroles
  - roles
  - rolebindings
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - bindings
  - events
  - limitranges
  - namespaces/status
  - pods/log
  - pods/status
  - replicationcontrollers/status
  - resourcequotas
  - resourcequotas/status
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - storage.k8s.io
  resources:
  - storageclasses
  - volumeattachments
  verbs:
  - get
  - list
  - watch
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: metrics
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-metrics
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: release-name-kubernetes-dashboard-metrics
subjects:
- kind: ServiceAccount
  name: release-name-kubernetes-dashboard-default
  namespace: default
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: readonly
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-readonly
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: release-name-kubernetes-dashboard-readonly
subjects:
- kind: ServiceAccount
  name: release-name-kubernetes-dashboard-default
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
rules:
- apiGroups:
  - ""
  resourceNames:
  - release-name-kubernetes-dashboard-settings
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resourceNames:
  - release-name-kubernetes-dashboard-certs
  - kubernetes-dashboard-csrf
  - kubernetes-dashboard-key-holder
  resources:
  - secrets
  verbs:
  - get
  - update
  - delete
- apiGroups:
  - ""
  resourceNames:
  - heapster
  - dashboard-metrics-scraper
  resources:
  - services
  verbs:
  - proxy
- apiGroups:
  - ""
  resourceNames:
  - heapster
  - 'http:heapster:'
  - 'https:heapster:'
  - dashboard-metrics-scraper
  - http:dashboard-metrics-scraper
  resources:
  - services/proxy
  verbs:
  - get
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

## Wrap up

That is almost it for this tutorial part. If you want to compare the intended outcome with what you created, your `values.yaml` should look like this at this point:

```yml
metrics-server:
  enabled: false
hull:
  config:
    specific:
      rbac:
        clusterReadOnlyRole: false
        clusterRoleMetrics: true
      settings: {}
      pinnedCRDs: {}
  objects:
    role:
      default:
        rules:
          secrets:
            apiGroups:
            - ""
            resources:
            - secrets
            resourceNames: _HT![
              {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "certs") }},
              "kubernetes-dashboard-csrf",
              "kubernetes-dashboard-key-holder"
              ]
            verbs:
            - get
            - update
            - delete
          configmaps:
            apiGroups:
            - ""
            resources:
            - configmaps
            resourceNames: _HT![
              {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "settings") }} ]
            verbs:
            - get
            - update
          services:
            apiGroups:
            - ""
            resources:
            - services
            resourceNames:
            - "heapster"
            - "dashboard-metrics-scraper"
            verbs:
            - proxy
          services_proxy:
            apiGroups:
            - ""
            resources:
            - services/proxy
            resourceNames:
            - "heapster"
            - "http:heapster:"
            - "https:heapster:"
            - "dashboard-metrics-scraper"
            - "http:dashboard-metrics-scraper"
            verbs:
            - get
    clusterrole:
      readonly:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterReadOnlyRole
        rules:
          cluster_objects:
            apiGroups:
            - ""
            resources:
            - configmaps
            - endpoints
            - persistentvolumeclaims
            - pods
            - replicationcontrollers
            - replicationcontrollers/scale
            - serviceaccounts
            - services
            - nodes
            - persistentvolumeclaims
            - persistentvolumes
            verbs:
            - get
            - list
            - watch
          statuses:
            apiGroups:
            - ""
            resources:
            - bindings
            - events
            - limitranges
            - namespaces/status
            - pods/log
            - pods/status
            - replicationcontrollers/status
            - resourcequotas
            - resourcequotas/status
            verbs:
            - get
            - list
            - watch
          namespaces:
            apiGroups:
            - ""
            resources:
            - namespaces
            verbs:
            - get
            - list
            - watch
          apps:
            apiGroups:
            - apps
            resources:
            - daemonsets
            - deployments
            - deployments/scale
            - replicasets
            - replicasets/scale
            - statefulsets
            verbs:
            - get
            - list
            - watch
          autoscaling:
            apiGroups:
            - autoscaling
            resources:
            - horizontalpodautoscalers
            verbs:
            - get
            - list
            - watch
          jobs:
            apiGroups:
            - batch
            resources:
            - cronjobs
            - jobs
            verbs:
            - get
            - list
            - watch
          extensions:
            apiGroups:
            - extensions
            resources:
            - daemonsets
            - deployments
            - deployments/scale
            - ingresses
            - networkpolicies
            - replicasets
            - replicasets/scale
            - replicationcontrollers/scale
            verbs:
            - get
            - list
            - watch
          pdb:
            apiGroups:
            - policy
            resources:
            - poddisruptionbudgets
            verbs:
            - get
            - list
            - watch
          network:
            apiGroups:
            - networking.k8s.io
            resources:
            - networkpolicies
            - ingresses
            verbs:
            - get
            - list
            - watch
          storage:
            apiGroups:
            - storage.k8s.io
            resources:
            - storageclasses
            - volumeattachments
            verbs:
            - get
            - list
            - watch
          rbac:
            apiGroups:
            - rbac.authorization.k8s.io
            resources:
            - clusterrolebindings
            - clusterroles
            - roles
            - rolebindings
            verbs:
            - get
            - list
            - watch
      metrics:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterRoleMetrics
        rules:
          metrics:
            apiGroups:
            - "metrics.k8s.io"
            resources:
            - pods
            - nodes
            verbs:
            - get
            - list
            - watch
    clusterrolebinding:
      readonly:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterReadOnlyRole
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^readonly
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^default
      metrics:
        enabled: _HT?(index . "$").Values.hull.config.specific.rbac.clusterRoleMetrics
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^metrics
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^default
```

Backup your finished `values.yaml` to the `values.tutorial-part.yaml` in case you want to modify, play around or start from scratch again with the `values.yaml`:

```sh
cp values.yaml values.tutorial-part.yaml
```

Lastly, we combine our previous results with the newly written ones. For this add the intermediate results to the `values.full.yaml`. Since we copied the `hull.config.specific` part in the beginning of this tutorial part we can just overwrite it with the one we expanded upon during this tutorial part. But the already existing `hull.objects` need to be merged with the newly defined ones so you can copy them first from `values.full.yaml` before appending them to the `values.yaml` content and saving the complete result as `values.full.yaml` again:

```sh
sed '1,/objects:/d' values.full.yaml > _tmp && cp values.yaml values.full.yaml && cat _tmp >> values.full.yaml && rm _tmp
```

The result of this course can be downloaded [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/04_rbac) for reference. 

__See you on the [next part of this series hopefully where you take a close look at defining workload objects via HULL](https://dev.to/gre9ory/hull-tutorial-05-defining-workload-objects-268l). Thanks for reading!__
