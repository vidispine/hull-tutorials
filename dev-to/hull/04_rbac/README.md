After the ConfigMaps and Secrets tutorial part this part will be used to look at the RBAC configuration side of `ingress-nginx`'s Helm chart and check how it is transportable to our new HULL based `ingress-nginx-hull` chart. 

Initially we copy over our working chart from the last tutorial part to a new folder in the same manner as we did before. Go to your working directory and execute:

```bash
$ cp -R 03_configmaps_and_secrets/ 04_rbac && cd 04_rbac/ingress-nginx-hull
```

From the `values.full.yaml`, the starting point for this tutorial part, only keep the `hull.config.specific` section for building upon in this chapter, the `hull.objects` section can be cleared for now since you will only add objects relevant to this tutorial section to it. At the end of this tutorial part the newly created objects and `hull.config.specific` properties are combined in the `values.intermediate.yaml` with what we already have created in which will then serve as the next tutorial parts starting point. 

To delete the HULL objects from `values.yaml` type:

```bash
$ sed -i '/  objects:/Q' values.yaml
```

and verify the result:

```bash
$ cat values.yaml
```

```yaml
hull:
  config:
    specific:
      controller:
        configMaps:
          tcp:
            mappings: {}
          udp:
            mappings: {}
```

Now you ar ready to proceed!

## Examining the existing RBAC objects

Execute the following command to first get an idea which RBAC objects are defined:

```bash
$ find ../ingress-nginx/templates -type f -iregex '.*\(Role\|\role|\Account\|account\|rbac\|RBAC\).*' | sort
```
```yaml
../ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/clusterrolebinding.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/serviceaccount.yaml
../ingress-nginx/templates/clusterrole.yaml
../ingress-nginx/templates/clusterrolebinding.yaml
../ingress-nginx/templates/controller-role.yaml
../ingress-nginx/templates/controller-rolebinding.yaml
../ingress-nginx/templates/controller-serviceaccount.yaml
../ingress-nginx/templates/default-backend-role.yaml
../ingress-nginx/templates/default-backend-rolebinding.yaml
../ingress-nginx/templates/default-backend-serviceaccount.yaml
```

In the previous tutorial part when you created the `ingress-nginx-hull` chart you already had a glance at the default RBAC configuration that comes with HULL. So now is the time to adapt and tweak it so it matches the original Helm charts RBAC functionally as closely as possible.

First check the `controller` RBAC-related ServiceAccount, Role and RoleBinding objects starting with the ServiceAccount:

```bash
$ cat ../ingress-nginx/templates/controller-serviceaccount.yaml
```
```yaml
{{- if or .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ template "ingress-nginx.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
automountServiceAccountToken: {{ .Values.serviceAccount.automountServiceAccountToken }}
{{- end }}
```

The actual creation of the ServiceAccount name is deferred to a helper function in `_helpers.tpl`:

```bash
$ cat ../ingress-nginx/templates/_helpers.tpl | grep 'ingress-nginx\.serviceAccountName' -B 10 -A 10
```
```yaml
Selector labels
*/}}
{{- define "ingress-nginx.selectorLabels" -}}
app.kubernetes.io/name: {{ include "ingress-nginx.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
{{/*
Create the name of the controller service account to use
*/}}
{{- define "ingress-nginx.serviceAccountName" -}}
{{- if .Values.serviceAccount.create -}}
    {{ default (include "ingress-nginx.fullname" .) .Values.serviceAccount.name }}
{{- else -}}
    {{ default "default" .Values.serviceAccount.name }}
{{- end -}}
{{- end -}}

{{/*
Create the name of the backend service account to use - only used when podsecuritypolicy is also enabled
*/}}
```

Lastly check the `values.yaml` section dealing with the `serviceAccount` configuration:

```bash
$ cat ../ingress-nginx/values.yaml | grep '^serviceAccount' -B 10 -A 10
```
```yaml
## Enable RBAC as per https://github.com/kubernetes/ingress-nginx/blob/main/docs/deploy/rbac.md and https://github.com/kubernetes/ingress-nginx/issues/266
rbac:
  create: true
  scope: false
# If true, create & use Pod Security Policy resources
# https://kubernetes.io/docs/concepts/policy/pod-security-policy/
podSecurityPolicy:
  enabled: false
serviceAccount:
  create: true
  name: ""
  automountServiceAccountToken: true
## Optional array of imagePullSecrets containing private registry credentials
## Ref: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
imagePullSecrets: []
# - name: secretName
# TCP service key:value pairs
```

Now you should have gathered an impression of the ServiceAccount configuration options and relation so let's look at how HULL handles this aspect.

## Choosing ServiceAccounts for pods in HULL

When breaking down the execution logic that creates the name of the ServiceAccount just examined, the following four variants chrystalize themselves (in _cursive_ the conditions that apply to each variant): 

1. create a new individual ServiceAccount for this chart named as the charts 'fullname' as `<release-name>-<chart-name>`.

    _Settingwise this is done by enabling `serviceAccount.create`  and not providing a `serviceAccount.name`)._ 

    **This is the default in `ingress-nginx`'s `values.yaml`.**

2. create a new individual ServiceAccount for this chart with a fixed provided name. 

    _Achieved by setting `serviceAccount.create` to `true` and providing a `serviceAccount.name`._

3. use a given ServiceAccount with a fixed provided name. 

    _For this choice set `serviceAccount.create` to `false` and prove the full `serviceAccount.name`_

4. use the Kubernetes default ServiceAccount simply named 'default'. 

    _setting `serviceAccount.create` to `false` and not supplying a `serviceAccount.name` is required_

Now within HULL the ServiceAccount for a pod is derived generally by the following rules:

1. if no specific `serviceAccountName` is set on the pod it is checked whether `hull.objects.serviceaccount.default.enabled` is `true` (defaults to `true`) and if so the chart and release specific default ServiceAccount is being used named `<release-name>-<chart-name>-default`. A matching `default` Role and RoleBinding is also created by default.

    **This is the default in HULL's `values.yaml`**

2. if a specific `serviceAccountName` was set on the pod it is used for that pod. Note that this field requires the full name to be set so in case you want to create and use a dynamically named ServiceAccount you need to use the fullname transformation (short form) from HULL to generate this name with `serviceAccountName: _HT^my-components-name` which would produce `serviceAccountName: <release-name>-<chart-name>-my-components-name` in the rendered output.

3. lastly if the release specific default ServiceAccount is disabled, the property `serviceAccountName` is not set which will default in Kubernetes to using the Kubernetes standard (simply named 'default') ServiceAccount.

### Using the release specific default ServiceAccount

Comparing the logics between the `ingress-nginx` and HULL based ServiceAccount handling you can first see that the default with HULL is very similar to the default in the original chart (case 1.). The only minor exception here is that the `-default` suffix is added to the newly created ServiceAccount by HULL. 

Using a test file named `../configs/default-sa.yaml` you can produce this default case for the roughly sketched `controller` Deployment which just has a pod spec for now. 

> Workloads such as the `controller` Deployment will be part of a later tutorial so you can only concentrate on the `serviceAccountName` property setting for now.

```bash
$ echo 'hull:
  objects:
    deployment:
      controller:
        pod:
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
' > ../configs/default-sa.yaml
```

Apply and check:

```bash
$ helm template -f ../configs/default-sa.yaml .
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
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-controller
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: RELEASE-NAME
      app.kubernetes.io/name: ingress-nginx-hull
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - env: []
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        name: controller
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
```

As seen from the output, `serviceAccountName: release-name-ingress-nginx-hull-default` is used for the pod by default which is as intended. The `default` Role and RoleBindings are available for RBAC configuration. 

The other original cases 2.-4. can be realized with HULL too by reconfiguring the chart at deploy time. 

### Using the Kubernetes 'default' ServiceAccount

When setting `hull.objects.serviceaccount.default.enabled` to `false` and not setting any `serviceAccountName` on pods, the Kubernetes standard ('default') account is used for all pods (matching original case 4.). If you don't want to have or use the `default` ServiceAccount at all in your applications Helm Chart but still use RBAC it makes sense to disable the `default` Role and RoleBindings as well so they do not point at a nonexisting ServiceAccount. Here we explicitly disable them in the upcoming examples if we don't need them for the example. Also, if you don't use RBAC at all, disabling the `hull.config.specific.rbac` setting will also prevent rendering of Roles and RoleBindings.

Create a pod using the standard Kubernetes 'default' ServiceAccount like this:

```bash
$ echo 'hull:
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
      controller:
        pod:
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
' > ../configs/k8s-default-sa.yaml
```

Again check the result:

```bash
$ helm template -f ../configs/k8s-default-sa.yaml .
```
```yaml
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-controller
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: RELEASE-NAME
      app.kubernetes.io/name: ingress-nginx-hull
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - env: []
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        name: controller
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      volumes: []
```

No `default` ServiceAccount, Role or RoleBinding was created and no ServiceAccount is set on the pod so the 'default' one from Kubernetes is applied as intended. 

### Using a ServiceAccount defined outside of Helm Chart

To produce the original case 3 setting explicit `serviceAccountName`'s on individual pods does the trick. It is up to you if you want to set the `default` ServiceAccount, Role or RoleBinding to disabled but let's do it here since we don't plan on using them elsewhere in this example:

```bash
$ echo 'hull:
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
      controller:
        pod:
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
          serviceAccountName: my-external-cluster-sa
' > ../configs/external-sa.yaml
```

Observe the output:

```bash
$ helm template -f ../configs/external-sa.yaml .
```
```yaml
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-controller
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: RELEASE-NAME
      app.kubernetes.io/name: ingress-nginx-hull
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - env: []
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        name: controller
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: my-external-cluster-sa
      volumes: []
```

The static name for the ServiceAccount was used. If you manage the ServiceAccounts outside of the Helm chart this is a typical scenario.

### Using release specific non-default ServiceAccounts

Last case to cover is case 2. To emulate this, setting `hull.objects.serviceaccount.default.enabled` to `false` and creating one or more ServiceAccounts for RBAC assignment with the desired names is required. 

In terms of choosing the name for a ServiceAccount to be created and used in the pod spec there are two choices which are both coverable by HULL:

#### Specifying unique and non-default dynamic ServiceAccounts

To create a dynamic name (with pattern `<chart-name>-<release-name>-<component-name>`) it is required to apply a transformation on the `serviceAccountName` on the individual pods to include the dynamic prefix in the rendered result like in this example:

```bash
$ echo 'hull:    
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
      controller:
        pod:
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
          serviceAccountName: _HT^other_sa
' > ../configs/internal-dynamic-sa.yaml
``` 
```bash
$ helm template -f ../configs/internal-dynamic-sa.yaml .
```
```yaml
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: other_sa
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-other_sa
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-controller
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: RELEASE-NAME
      app.kubernetes.io/name: ingress-nginx-hull
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - env: []
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        name: controller
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: release-name-ingress-nginx-hull-other_sa
      volumes: []
```

As you can see the name of the created ServiceAccount `name: release-name-ingress-nginx-hull-other_sa` matches the pods `serviceAccountName: release-name-ingress-nginx-hull-other_sa` exactly as intended. In this scenario you may want to proceed defining Role and RoleBindings as required for the `other_sa` ServiceAccount.

#### Specifying non-default static ServiceAccounts

The other alternative is to create a completely static ServiceAccount name (without the `<chart-name>-<release-name>-` prefix) and use this for the pod(s). For this you do not need the `makefullname` transformation of the pods `serviceAccountName` but just add the `staticName: true` property to the ServiceAccount object:

```bash
$ echo 'hull:    
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
      controller:
        pod:
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
          serviceAccountName: other_sa
' > ../configs/internal-static-sa.yaml
``` 

Check the result:

```bash
$ helm template -f ../configs/internal-static-sa.yaml .
```
```yaml
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: v1
automountServiceAccountToken: true
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: other_sa
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: other_sa
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-controller
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: controller
      app.kubernetes.io/instance: RELEASE-NAME
      app.kubernetes.io/name: ingress-nginx-hull
  template:
    metadata:
      annotations: {}
      labels:
        app.kubernetes.io/component: controller
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - env: []
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        name: controller
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: other_sa
      volumes: []
```

> The caveat to this approach is that you may need to choose carefully which name to create because it may already exist in the namespace as opposed to sticking with dynamic names only so that names are highly unique. If you choose a static name that already exists in the namespace that will cause an error at deploy time!

Except for the default case of creating a dynamic and unique default ServiceAccount, the other ServiceAccount handling cases require little additional configuration. However, the HULL based configuration is transparent, explicit and flexible if special needs exist and the full flexibility of ServiceAccount management is needed to adapt to a given scenario. Otherwise providing pre-defined RBAC objects for the `default` ServiceAccount and using this ServiceAccount in the pods is likely enough for smaller application deployments.

## Defining a RoleBinding

Next up is adding more RBAC objects to the mix: the `controller` RoleBinding:

```bash
$ cat ../ingress-nginx/templates/controller-rolebinding.yaml
```
```yaml
{{- if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ include "ingress-nginx.fullname" . }}
  namespace: {{ .Release.Namespace }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "ingress-nginx.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ template "ingress-nginx.serviceAccountName" . }}
    namespace: {{ .Release.Namespace | quote }}
{{- end }}
```

In HULL RBAC is controlled by a global RBAC `true`/`false` switch which is built into HULL as a predefined key at `hull.config.general.rbac`. If `true`, the RBAC related objects are rendered, if false they are just skipped. This mimics the behaviour that is very common in many popular Helm charts. 

> The recommendation is to always leave RBAC enabled in your clusters.

From the above `cat` output you can see that the RoleBinding and referenced Role name are again dependent on the `ingress-nginx.fullname` function and the ServiceAccount name is subject to the `ingress-nginx.serviceAccountName` function examined earlier. Unless you need to or want to change the controllers ServiceAccount name from the default there is nothing to do here except having RBAC enabled to create a sufficient `default` RoleBinding in HULL looking like this:

```yaml
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

If you want to deviate from the default handling you can freely change the `roleRef` and `subjects` as needed, the same techniques already explained above do apply for the creation of object names.

## Configuring Roles

Now for the `controller`'s Role:

```bash
$ cat ../ingress-nginx/templates/controller-role.yaml
```
```yaml
{{- if .Values.rbac.create -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ include "ingress-nginx.fullname" . }}
  namespace: {{ .Release.Namespace }}
rules:
  - apiGroups:
      - ""
    resources:
      - namespaces
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - configmaps
      - pods
      - secrets
      - endpoints
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      - {{ .Values.controller.electionID }}
    verbs:
      - get
      - update
  - apiGroups:
      - ""
    resources:
      - configmaps
    verbs:
      - create
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
{{- if .Values.podSecurityPolicy.enabled }}
  - apiGroups:      [{{ template "podSecurityPolicy.apiGroup" . }}]
    resources:      ['podsecuritypolicies']
    verbs:          ['use']
    {{- with .Values.controller.existingPsp }}
    resourceNames:  [{{ . }}]
    {{- else }}
    resourceNames:  [{{ include "ingress-nginx.fullname" . }}]
    {{- end }}
{{- end }}
{{- end }}
```

The `default` Role that comes with HULL can be used for this in the standard scenario but needs adding of the specific `rules:` required. There are two interesting aspects about the given `rules:` section displayed above, namely the reference to `controller.electionID`:

```yaml
- apiGroups:
      - ""
    resources:
      - configmaps
    resourceNames:
      - {{ .Values.controller.electionID }}
    verbs:
      - get
      - update
```

and the PodSecurityPolicy part:

```yaml
{{- if .Values.podSecurityPolicy.enabled }}
  - apiGroups:      [{{ template "podSecurityPolicy.apiGroup" . }}]
    resources:      ['podsecuritypolicies']
    verbs:          ['use']
    {{- with .Values.controller.existingPsp }}
    resourceNames:  [{{ . }}]
    {{- else }}
    resourceNames:  [{{ include "ingress-nginx.fullname" . }}]
    {{- end }}
{{- end }}
```

A search reveals that the `controller.electionID` defaults to value `ingress-controller-leader` in the `values.yaml` and is also used in the composition of the container arguments for the `controller` Deployment and `controller` DaemonSet:

```bash
$ find ../ingress-nginx -type f -print | xargs grep "electionID"
```
```yaml
../ingress-nginx/templates/controller-role.yaml:      - {{ .Values.controller.electionID }}
../ingress-nginx/templates/controller-deployment.yaml:            - --election-id={{ .Values.controller.electionID }}
../ingress-nginx/templates/controller-daemonset.yaml:            - --election-id={{ .Values.controller.electionID }}
../ingress-nginx/values.yaml:  electionID: ingress-controller-leader
```

To define a common source for this field to which to reference to from the multiple places it's being used you should again use the HULL configuration section that is intended for this: `hull.config.specific`. There you can create a field named `hull.config.specific.controller.electionID` and reference to it from the `rules:` section (and later in the `controller` containers `args:`). You can do that in a minute but first the PodSecurityPolicy part needs modelling. 

There are two ways to approach the PodSecurityPolicy part modeling: either write out the complete `rules` section via a `tpl` transformation or define the individual rules in a key value style allowing the chart user to easily manipulate individual entries. As it was already shown how to create a complete dictionary using the `tpl` transformation this time you use the HULL key-value based approach.

## Using a key-value approach to HULL based properties

Many HULL based properties which overwrite defined standard Kubernetes properties were written to allow a key value definition on the HULL side that are converted to arrays on the Kubernetes side. The advantage for particular selected fields is that it becomes possible to address and overwrite individual array items using Helm, something that is otherwise not possible via Helm when working on arrays. Kubernetes itself offers merging capabilities for individual array items but Helm does generally not support this. In case an array requires modification at deploy time it normally needs to be redefined in its entirety which requires a lookup on what the default array state is in a charts `values.yaml`. 

Now the `rules` array in the `role` objects is such a HULL handled property and hence we can put a key to the individual entries that makes them targetable for selected modification. Write out the `rules` for the `default` Role in the key value style like this, already adding the needed `hull.config.specific` properties:

```bash
$ echo '        electionID: ingress-controller-leader
        existingPsp: 
      podSecurityPolicy:
        enabled: false
  objects:
    role:
      default:
        rules:
          namespaces:
            apiGroups:
            - ""
            resources:
            - namespaces
            verbs:
            - get
          core:
            apiGroups:
            - ""
            resources:
            - configmaps
            - pods
            - secrets
            - endpoints
            verbs:
            - get
            - list
            - watch
          services:
            apiGroups:
            - ""
            resources:
            - services
            verbs:
            - get
            - list
            - watch
          ingresses:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses
            verbs:
            - get
            - list
            - watch
          ingress_status:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses/status
            verbs:
            - update
          ingress_classes:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingressclasses
            verbs:
            - get
            - list
            - watch
          electionid:
            apiGroups:
            - ""
            resources:
            - configmaps
            resourceNames:
            - _HT![ {{ (index . "$").Values.hull.config.specific.controller.electionID }} ]
            verbs:
            - get
            - update
          configmaps:
            apiGroups:
            - ""
            resources:
            - configmaps
            verbs:
            - create
          events:
            apiGroups:
            - ""
            resources:
            - events
            verbs:
            - create
            - patch
          psp:
            enabled: _HT?(index . "$").Values.hull.config.specific.podSecurityPolicy.enabled
            apiGroups:      
            - policy
            resources:      
            - podsecuritypolicies
            verbs:
            - use
            resourceNames:
            - _HT![ {{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "controller")) (index . "$").Values.hull.config.specific.controller.existingPsp }} ]' >> values.yaml
```

To perform a sanity check you can use this external config to test the behavior when the `electionID` changes and `podSecurityPolicy` is enabled:

```bash
$ echo 'hull:
  config:
    specific:
      controller:
        electionID: another-id
        existingPsp: usable-psp
      podSecurityPolicy:
        enabled: true' > ../configs/electionid-psp.yaml
```
```bash
$ helm template -f ../configs/electionid-psp.yaml .
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
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resourceNames:
  - another-id
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
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
- apiGroups:
  - policy
  resourceNames:
  - usable-psp
  resources:
  - podsecuritypolicies
  verbs:
  - use
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
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

The result looks as expected from the applied configuration.

Now that you have covered the typical ServiceAccount, Role and RoleBinding trinity for the `controller` component, it is time to work on the  `default-backend` and `admission-webhooks/job-patch` RBAC components and ServiceAccounts:

Here is the `default-backend` ServiceAccount:

```bash
$ cat ../ingress-nginx/templates/default-backend-serviceaccount.yaml
```
```yaml
{{- if and .Values.defaultBackend.enabled  .Values.defaultBackend.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: default-backend
  name: {{ template "ingress-nginx.defaultBackend.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
automountServiceAccountToken: {{ .Values.defaultBackend.serviceAccount.automountServiceAccountToken }}
{{- end }}
```

and the RoleBinding:

```bash
$ cat ../ingress-nginx/templates/default-backend-rolebinding.yaml
```
```yaml
{{- if and .Values.rbac.create .Values.podSecurityPolicy.enabled .Values.defaultBackend.enabled -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: default-backend
  name: {{ include "ingress-nginx.fullname" . }}-backend
  namespace: {{ .Release.Namespace }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "ingress-nginx.fullname" . }}-backend
subjects:
  - kind: ServiceAccount
    name: {{ template "ingress-nginx.defaultBackend.serviceAccountName" . }}
    namespace: {{ .Release.Namespace | quote }}
{{- end }}
```

and the Role:

```bash
$ cat ../ingress-nginx/templates/default-backend-role.yaml
```
```yaml
{{- if and .Values.rbac.create .Values.podSecurityPolicy.enabled .Values.defaultBackend.enabled -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: default-backend
  name: {{ include "ingress-nginx.fullname" . }}-backend
  namespace: {{ .Release.Namespace }}
rules:
  - apiGroups:      [{{ template "podSecurityPolicy.apiGroup" . }}]
    resources:      ['podsecuritypolicies']
    verbs:          ['use']
    {{- with .Values.defaultBackend.existingPsp }}
    resourceNames:  [{{ . }}]
    {{- else }}
    resourceNames:  [{{ include "ingress-nginx.fullname" . }}-backend]
    {{- end }}
{{- end }}
```

All in all you can see it is very similar content as in the `controller` case but with some differences:

- a global option `defaultBackend.enable` to not create any objects associated with the 'defaultBackend'

- an additional dependency on a global `podSecurityPolicy.enabled` condition for creating the Role and RoleBinding

In terms of the ServiceAccount management the options are those already explained. The chart user can decide to use an existing ServiceAccount or create one to be used in this Helm Chart. It seems feasible however to model the default behavior expressed in our new chart by having a global switch to disable the `default-backend` objects and create the default ServiceAccount, Role and RoleBinding for the `defaultBackend` if it isn't disabled.

Now examine the `admission-webhooks/job-patch` RBAC and ServiceAccount objects:

```bash
$ cat ../ingress-nginx/templates/admission-webhooks/job-patch/serviceaccount.yaml
```
```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "ingress-nginx.fullname" . }}-admission
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
{{- end }}
```

```bash
$ cat ../ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
```
```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ include "ingress-nginx.fullname" . }}-admission
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ include "ingress-nginx.fullname" . }}-admission
subjects:
  - kind: ServiceAccount
    name: {{ include "ingress-nginx.fullname" . }}-admission
    namespace: {{ .Release.Namespace | quote }}
{{- end }}
```

```bash
$ cat ../ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
```
```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name:  {{ include "ingress-nginx.fullname" . }}-admission
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
rules:
  - apiGroups:
      - ""
    resources:
      - secrets
    verbs:
      - get
      - create
{{- end }}
```

The only new addition to what has been done sofar is a dependency here on two switches that need to be added to the `hull.config.specific`  (`hull.config.specific.controller.admissionWebhooks.enabled` and `hull.config.specific.controller.admissionWebhooks.patch.enabled`) and the addition of object specific annotations in form of Helm hooks which we can handle same as any added annotation.

Ok, add the central defaultBackend switches:

```bash
$ sed '/^\s\s\s\sspecific/r'<(
      echo "      defaultBackend:"
      echo "        enabled: false"
      echo "        existingPsp:"
    ) -i -- values.yaml
```

and the added central controller properties:

```bash
$ sed '/^\s\s\s\s\s\scontroller:/r'<(
      echo "        admissionWebhooks:"
      echo "          enabled: true"
      echo "          existingPsp:"
      echo "          patch:"
      echo "            enabled: true"
    ) -i -- values.yaml
```

after which the top of your `values.yaml` should be equivalent to this:

```yaml
hull:
  config:
    specific:
      defaultBackend:
        enabled: true
        existingPsp:
      controller:
        admissionWebhooks:
          enabled: true
          existingPsp:
          patch:
            enabled: true
        configMaps:
          tcp:
            mappings: {}
          udp:
            mappings: {}
        electionID: ingress-controller-leader
        existingPsp:
      podSecurityPolicy:
        enabled: false
  objects:
```

Then define the objects that make up the RBAC aspects of the `default-backend` and the `admission-webhooks/job-patch`:

```bash
$ echo '      defaultbackend:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.defaultBackend.enabled (index . "$").Values.hull.config.specific.podSecurityPolicy.enabled)
        rules:
          psp:
            apiGroups:
            - policy
            resources:
            - podsecuritypolicies
            verbs:
            - use
            resourceNames:
            - _HT![ {{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "defaultbackend")) (index . "$").Values.hull.config.specific.defaultBackend.existingPsp }} ]
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        rules:
          psp:
            apiGroups:
            - ""
            resources:
            - secrets
            verbs:
            - get
            - create
    serviceaccount:
      defaultbackend:  
        enabled: _HT?(index . "$").Values.hull.config.specific.defaultBackend.enabled
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    rolebinding:
      defaultbackend:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.defaultBackend.enabled (index . "$").Values.hull.config.specific.podSecurityPolicy.enabled)
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: _HT^defaultbackend
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^defaultbackend
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: _HT^admission
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^admission' >> values.yaml
```

Note that another transformation slipped in, the `hull.util.transformation.makefullname` transformations short name `_HT^`. This is a transformation shortcut to using `tpl` with content `include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "<COMPONENT>")` to generate a unique name.

Time to view the result with no external files applied:

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
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resourceNames:
  - ingress-controller-leader
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
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
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
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
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admission
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admission
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  verbs:
  - get
  - create
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade,post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admission
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admission
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: release-name-ingress-nginx-hull-admission
subjects:
- kind: ServiceAccount
  name: release-name-ingress-nginx-hull-admission
  namespace: default
```

As expected only Roles, RoleBindings and ServiceAccounts for the `default` and `admission` components are present.

For a quick check you can disable the `admissionWebhooks` and enable `podSecurityPolicy` and `defaultBackend`. The expected outcome is added Roles, RoleBindings and ServiceAccounts for the `defaultbackend` and missing `admission` objects:

```bash
$ echo 'hull:    
  config:
    specific:
      defaultBackend:
        enabled: true
      controller:
        admissionWebhooks:
          enabled: false
      podSecurityPolicy:
        enabled: true
' > ../configs/default-backend-psp.yaml
``` 

Applying it:

```bash
$ helm template -f ../configs/default-backend-psp.yaml .
``` 

gives only `default` and `defaultbackend` objects as desired:

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
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: defaultbackend
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-defaultbackend
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
rules:
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - create
- apiGroups:
  - ""
  resources:
  - configmaps
  - pods
  - secrets
  - endpoints
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resourceNames:
  - ingress-controller-leader
  resources:
  - configmaps
  verbs:
  - get
  - update
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingressclasses
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
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
- apiGroups:
  - policy
  resourceNames:
  - release-name-ingress-nginx-hull-controller
  resources:
  - podsecuritypolicies
  verbs:
  - use
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - get
  - list
  - watch
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: defaultbackend
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-defaultbackend
rules:
- apiGroups:
  - policy
  resourceNames:
  - release-name-ingress-nginx-hull-defaultbackend
  resources:
  - podsecuritypolicies
  verbs:
  - use
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
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: defaultbackend
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-defaultbackend
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: release-name-ingress-nginx-hull-defaultbackend
subjects:
- kind: ServiceAccount
  name: release-name-ingress-nginx-hull-defaultbackend
  namespace: defaultyaml
```

On to the last aspect of this tutorial part, the cluster-level RBAC objects.

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

First convert the ClusterRoleBindings from the `/templates/admission-webhooks/job-patch` folder:

```bash
$  cat ../ingress-nginx/templates/admission-webhooks/job-patch/clusterrolebinding.yaml
``` 
```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name:  {{ include "ingress-nginx.fullname" . }}-admission
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "ingress-nginx.fullname" . }}-admission
subjects:
  - kind: ServiceAccount
    name: {{ include "ingress-nginx.fullname" . }}-admission
    namespace: {{ .Release.Namespace | quote }}
{{- end }}
```

and from the `/templates` folder:

```bash
$  cat ../ingress-nginx/templates/clusterrolebinding.yaml
``` 
```yaml
{{- if and .Values.rbac.create (not .Values.rbac.scope) -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
  name: {{ include "ingress-nginx.fullname" . }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: {{ include "ingress-nginx.fullname" . }}
subjects:
  - kind: ServiceAccount
    name: {{ template "ingress-nginx.serviceAccountName" . }}
    namespace: {{ .Release.Namespace | quote }}
{{- end }}
```

Adding them to the `values.yaml` should be mostly transparent now. One aspect to remark is the `.Values.rbac.scope` switch which toggles between a scoped RBAC solution (without ClusterRoles and ClusterRoleBindings) and an unscoped one utilizing the cluster level RBAC components.

So first add in the new "global" parameter:

```bash
$ sed '/^\s\s\s\sspecific/r'<(
      echo "      rbac:"
      echo "        scope: false"
    ) -i -- values.yaml
```

and then add the ClusterRoleBindings:

```bash
$ echo '    clusterrolebinding:
      default:
        enabled: _HT?(not (index . "$").Values.hull.config.specific.rbac.scoped)
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^default
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^default
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^admission
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^admission' >> values.yaml 
```

Easy wasn't it? Lastly check out the `templates/clusterrole.yaml` and `/templates/admission-webhooks/job-patch/clusterrole.yaml` files:

```bash
$ cat ../ingress-nginx/templates/clusterrole.yaml
```
```yaml
{{- if .Values.rbac.create }}

{{- if and .Values.rbac.scope (not .Values.controller.scope.enabled) -}}
  {{ required "Invalid configuration: 'rbac.scope' should be equal to 'controller.scope.enabled' (true/false)." (index (dict) ".") }}
{{- end }}

{{- if not .Values.rbac.scope -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
  name: {{ include "ingress-nginx.fullname" . }}
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - endpoints
      - nodes
      - pods
      - secrets
    verbs:
      - list
      - watch
{{- if and .Values.controller.scope.enabled .Values.controller.scope.namespace }}
  - apiGroups:
      - ""
    resources:
      - namespaces
    resourceNames:
      - "{{ .Values.controller.scope.namespace }}"
    verbs:
      - get
{{- end }}
  - apiGroups:
      - ""
    resources:
      - nodes
    verbs:
      - get
  - apiGroups:
      - ""
    resources:
      - services
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - events
    verbs:
      - create
      - patch
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingresses/status
    verbs:
      - update
  - apiGroups:
      - networking.k8s.io
    resources:
      - ingressclasses
    verbs:
      - get
      - list
      - watch
{{- end }}

{{- end }}
```

```bash
$ cat ../ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
```
```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: {{ include "ingress-nginx.fullname" . }}-admission
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
rules:
  - apiGroups:
      - admissionregistration.k8s.io
    resources:
      - validatingwebhookconfigurations
    verbs:
      - get
      - update
{{- if .Values.podSecurityPolicy.enabled }}
  - apiGroups: ['extensions']
    resources: ['podsecuritypolicies']
    verbs:     ['use']
    resourceNames:
    {{- with .Values.controller.admissionWebhooks.existingPsp }}
    - {{ . }}
    {{- else }}
    - {{ include "ingress-nginx.fullname" . }}-admission
    {{- end }}
{{- end }}
{{- end }}
```

> Note that the [Helm functionality to `fail` a configuration or to have `required` fields](https://austindewey.com/2018/12/28/helm-tricks-input-validation-with-required-and-fail/) is not supported by the way HULL processes the `values.yaml`. This is being used here in the `templates/clusterrole.yaml` case.

The `controller.scope` global section referenced in the `templates` ClusterRoleBinding is new and we add it first:

```bash
$ sed '/^\s\s\s\s\s\scontroller:/r'<(
      echo "        scope:"
      echo "          enabled: false"
      echo "          namespace: \"\""
    ) -i -- values.yaml
```

Then here is the added configuration to cover both ClusterRoles with the `ingress-nginx-hull` chart:

```bash
$ echo '    clusterrole:
      default:
        enabled: _HT?(not (index . "$").Values.hull.config.specific.rbac.scoped)
        rules:
          core:
            apiGroups:
            - ""
            resources:
            - configmaps
            - endpoints
            - nodes
            - pods
            - secrets
            verbs:
            - list
            - watch
          namespace:
            enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.scope.enabled (index . "$").Values.hull.config.specific.controller.scope.namespace)
            apiGroups:
            - ""
            resources:
            - namespaces
            resourceNames: 
            - _HT![ {{ (index . "$").Release.Namespace }} ]
            verbs:
            - get
          nodes:
            apiGroups:
            - ""
            resources:
            - nodes
            verbs:
            - get
          services:
            apiGroups:
            - ""
            resources:
            - services
            verbs:
            - get
            - list
            - watch
          ingresses:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses
            verbs:
            - get
            - list
            - watch
          ingress_status:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses/status
            verbs:
            - update
          ingress_classes:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingressclasses
            verbs:
            - get
            - list
            - watch
          electionid:
            apiGroups:
            - ""
            resources:
            - configmaps
            resourceNames:
            - _HT![ {{ (index . "$").Values.hull.config.specific.controller.electionID }} ]
            verbs:
            - get
            - update
          events:
            apiGroups:
            - ""
            resources:
            - events
            verbs:
            - create
            - patch
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        rules:
          admission:
            apiGroups:      
            - admissionregistration.k8s.io
            resources:      
            - validatingwebhookconfigurations
            verbs:
            - get
            - update
          psp:
            enabled: _HT?(index . "$").Values.hull.config.specific.podSecurityPolicy.enabled
            apiGroups:      
            - extensions
            resources:      
            - podsecuritypolicies
            verbs:
            - use
            resourceNames:
            - _HT![ {{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "controller")) (index . "$").Values.hull.config.specific.controller.admissionWebhooks.existingPsp }} ]' >> values.yaml
```

## Wrap up

That is almost it for this tutorial part. If you want to compare it with what you created, your `values.yaml` should look like this:

```yaml
hull:
  config:
    specific:
      rbac:
        scope: false
      defaultBackend:
        enabled: false
        existingPsp:
      controller:
        scope:
          enabled: false
          namespace: ""
        admissionWebhooks:
          enabled: true
          existingPsp:
          patch:
            enabled: true
        configMaps:
          tcp:
            mappings: {}
          udp:
            mappings: {}
        electionID: ingress-controller-leader
        existingPsp:
      podSecurityPolicy:
        enabled: false
  objects:
    role:
      default:
        rules:
          namespaces:
            apiGroups:
            - ""
            resources:
            - namespaces
            verbs:
            - get
          core:
            apiGroups:
            - ""
            resources:
            - configmaps
            - pods
            - secrets
            - endpoints
            verbs:
            - get
            - list
            - watch
          services:
            apiGroups:
            - ""
            resources:
            - services
            verbs:
            - get
            - list
            - watch
          ingresses:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses
            verbs:
            - get
            - list
            - watch
          ingress_status:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses/status
            verbs:
            - update
          ingress_classes:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingressclasses
            verbs:
            - get
            - list
            - watch
          electionid:
            apiGroups:
            - ""
            resources:
            - configmaps
            resourceNames:
            - _HT![ {{ (index . "$").Values.hull.config.specific.controller.electionID }} ]
            verbs:
            - get
            - update
          configmaps:
            apiGroups:
            - ""
            resources:
            - configmaps
            verbs:
            - create
          events:
            apiGroups:
            - ""
            resources:
            - events
            verbs:
            - create
            - patch
          psp:
            enabled: _HT?(index . "$").Values.hull.config.specific.podSecurityPolicy.enabled
            apiGroups:
            - policy
            resources:
            - podsecuritypolicies
            verbs:
            - use
            resourceNames:
            - _HT![ {{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "controller")) (index . "$").Values.hull.config.specific.controller.existingPsp }} ]
      defaultbackend:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.defaultBackend.enabled (index . "$").Values.hull.config.specific.podSecurityPolicy.enabled)
        rules:
          psp:
            apiGroups:
            - policy
            resources:
            - podsecuritypolicies
            verbs:
            - use
            resourceNames:
            - _HT![ {{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "defaultbackend")) (index . "$").Values.hull.config.specific.defaultBackend.existingPsp }} ]
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        rules:
          psp:
            apiGroups:
            - ""
            resources:
            - secrets
            verbs:
            - get
            - create
    serviceaccount:
      defaultbackend:
        enabled: _HT?(index . "$").Values.hull.config.specific.defaultBackend.enabled
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
    rolebinding:
      defaultbackend:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.defaultBackend.enabled (index . "$").Values.hull.config.specific.podSecurityPolicy.enabled)
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: _HT^defaultbackend
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^defaultbackend
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: _HT^admission
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^admission

    clusterrolebinding:
      default:
        enabled: _HT?(not (index . "$").Values.hull.config.specific.rbac.scoped)
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^default
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^default
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: ClusterRole
          name: _HT^admission
        subjects:
        - namespace: _HT!{{ (index . "$").Release.Namespace }}
          kind: ServiceAccount
          name: _HT^admission
    clusterrole:
      default:
        enabled: _HT?(not (index . "$").Values.hull.config.specific.rbac.scoped)
        rules:
          core:
            apiGroups:
            - ""
            resources:
            - configmaps
            - endpoints
            - nodes
            - pods
            - secrets
            verbs:
            - list
            - watch
          namespace:
            enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.scope.enabled (index . "$").Values.hull.config.specific.controller.scope.namespace)
            apiGroups:
            - ""
            resources:
            - namespaces
            resourceNames:
            - _HT![ {{ (index . "$").Release.Namespace }} ]
            verbs:
            - get
          nodes:
            apiGroups:
            - ""
            resources:
            - nodes
            verbs:
            - get
          services:
            apiGroups:
            - ""
            resources:
            - services
            verbs:
            - get
            - list
            - watch
          ingresses:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses
            verbs:
            - get
            - list
            - watch
          ingress_status:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingresses/status
            verbs:
            - update
          ingress_classes:
            apiGroups:
            - networking.k8s.io
            resources:
            - ingressclasses
            verbs:
            - get
            - list
            - watch
          electionid:
            apiGroups:
            - ""
            resources:
            - configmaps
            resourceNames:
            - _HT![ {{ (index . "$").Values.hull.config.specific.controller.electionID }} ]
            verbs:
            - get
            - update
          events:
            apiGroups:
            - ""
            resources:
            - events
            verbs:
            - create
            - patch
      admission:
        enabled: _HT?(and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled)
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade,post-install,post-upgrade
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        rules:
          admission:
            apiGroups:
            - admissionregistration.k8s.io
            resources:
            - validatingwebhookconfigurations
            verbs:
            - get
            - update
          psp:
            enabled: _HT?(index . "$").Values.hull.config.specific.podSecurityPolicy.enabled
            apiGroups:
            - extensions
            resources:
            - podsecuritypolicies
            verbs:
            - use
            resourceNames:
            - _HT![ {{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "controller")) (index . "$").Values.hull.config.specific.controller.admissionWebhooks.existingPsp }} ]
```

If you think this is long think about how much of the original charts templates and `values.yaml` logic is actually contained in there.

Back up your `values.yaml` to the `values.intermediate.yaml` in case you want to modify, play around or start from scratch again with the `values.yaml`:

```bash
$ cp values.yaml values.intermediate.yaml
```

Lastly, we combine our previous results with the newly written ones. For this add our 'intermediate' results to the `values.full.yaml`. Since we copied the `hull.config.specific` part in the beginning of this tutorial part we can just overwrite it with the one we expanded here. But the already existing `hull.objects` need to be merged with the new ones so you can copy them first from `values.full.yaml` before appending them to the `values.yaml` content and saving the result as `values.full.yaml` again:

```bash
$ sed '1,/objects:/d' values.full.yaml
 > _tmp && cp values.yaml values.full.yaml && cat _tmp >> values.full.yaml && rm _tmp
```

The result of this endavour can be downloaded [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/04_rbac) for reference. 

__Thanks for reading!__