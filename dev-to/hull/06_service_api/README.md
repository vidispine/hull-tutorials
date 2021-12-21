
## Preparation

In this Part of the tutorial workload objects are created. A workload object has a pod at its core in which containers run code. Most important workload types are Deployments, DaemonSets, StatefulSets, Jobs and CronJobs. 

First off we again copy over our working chart from the last tutorial part to a new folder in the same manner as we did before. Go to your working directory and execute:

```bash
$ cp -R 04_rbac/ 05_workloads && cd 05_workloads/ingress-nginx-hull
```
As before, to delete the HULL `hull.objects` from `values.full.yaml` and copy the current `hull.config` to the `values.yaml` type the following:

```bash
$ sed '/  objects:/Q' values.full.yaml > values.yaml
```

and verify the result:

```bash
$ cat values.yaml
```

looks like: 

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
```

If yes, then you are good to start the next tutorial part.

## Examining existing workloads

Now having the preparation out of the way first find out which of those mentioned workload objects are present in the original chart:

```bash
$ find ../ingress-nginx/templates -type f -iregex '.*\(service\).*' | sort
```

which returns:

```yaml
../ingress-nginx/templates/admission-webhooks/job-patch/clusterrole.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/clusterrolebinding.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/job-patchWebhook.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/psp.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/role.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/rolebinding.yaml
../ingress-nginx/templates/admission-webhooks/job-patch/serviceaccount.yaml
../ingress-nginx/templates/controller-daemonset.yaml
../ingress-nginx/templates/controller-deployment.yaml
../ingress-nginx/templates/default-backend-deployment.yaml
```

Two jobs, two deployments and one DaemonSet to look at.

### Jobs in HULL

Start with the `templates/admission-webhooks/job-patch/job-createSecret.yaml` Job:

```bash
$ cat ../ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
```

which is defined as follows:

```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "ingress-nginx.fullname" . }}-admission-create
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
spec:
{{- if .Capabilities.APIVersions.Has "batch/v1alpha1" }}
  # Alpha feature since k8s 1.12
  ttlSecondsAfterFinished: 0
{{- end }}
  template:
    metadata:
      name: {{ include "ingress-nginx.fullname" . }}-admission-create
    {{- if .Values.controller.admissionWebhooks.patch.podAnnotations }}
      annotations: {{ toYaml .Values.controller.admissionWebhooks.patch.podAnnotations | nindent 8 }}
    {{- end }}
      labels:
        {{- include "ingress-nginx.labels" . | nindent 8 }}
        app.kubernetes.io/component: admission-webhook
    spec:
    {{- if .Values.controller.admissionWebhooks.patch.priorityClassName }}
      priorityClassName: {{ .Values.controller.admissionWebhooks.patch.priorityClassName }}
    {{- end }}
    {{- if .Values.imagePullSecrets }}
      imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
    {{- end }}
      containers:
        - name: create
          {{- with .Values.controller.admissionWebhooks.patch.image }}
          image: "{{- if .repository -}}{{ .repository }}{{ else }}{{ .registry }}/{{ .image }}{{- end -}}:{{ .tag }}{{- if (.digest) -}} @{{.digest}} {{- end -}}"
          {{- end }}
          imagePullPolicy: {{ .Values.controller.admissionWebhooks.patch.image.pullPolicy }}
          args:
            - create
            - --host={{ include "ingress-nginx.controller.fullname" . }}-admission,{{ include "ingress-nginx.controller.fullname" . }}-admission.$(POD_NAMESPACE).svc
            - --namespace=$(POD_NAMESPACE)
            - --secret-name={{ include "ingress-nginx.fullname" . }}-admission
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          {{- if .Values.controller.admissionWebhooks.createSecretJob.resources }}
          resources: {{ toYaml .Values.controller.admissionWebhooks.createSecretJob.resources | nindent 12 }}
          {{- end }}
      restartPolicy: OnFailure
      serviceAccountName: {{ include "ingress-nginx.fullname" . }}-admission
    {{- if .Values.controller.admissionWebhooks.patch.nodeSelector }}
      nodeSelector: {{ toYaml .Values.controller.admissionWebhooks.patch.nodeSelector | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.admissionWebhooks.patch.tolerations }}
      tolerations: {{ toYaml .Values.controller.admissionWebhooks.patch.tolerations | nindent 8 }}
    {{- end }}
      securityContext:
        runAsNonRoot: true
        runAsUser: {{ .Values.controller.admissionWebhooks.patch.runAsUser }}
{{- end }}
```

Many Helm charts are full of 1:1 mappings between one `values.yaml` property and one single object's property in the `/templates` files. Cases in point for the above Job are:

```yaml
    {{- if .Values.controller.admissionWebhooks.patch.podAnnotations }}
      annotations: {{ toYaml .Values.controller.admissionWebhooks.patch.podAnnotations | nindent 8 }}
    {{- end }}
```
```yaml
    {{- if .Values.controller.admissionWebhooks.patch.priorityClassName }}
      priorityClassName: {{ .Values.controller.admissionWebhooks.patch.priorityClassName }}
    {{- end }}
```
```yaml
    {{- if .Values.imagePullSecrets }}
      imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
    {{- end }}
```
```yaml
          {{- if .Values.controller.admissionWebhooks.createSecretJob.resources }}
          resources: {{ toYaml .Values.controller.admissionWebhooks.createSecretJob.resources | nindent 12 }}
          {{- end }}
```
```yaml
    {{- if .Values.controller.admissionWebhooks.patch.nodeSelector }}
      nodeSelector: {{ toYaml .Values.controller.admissionWebhooks.patch.nodeSelector | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.admissionWebhooks.patch.tolerations }}
      tolerations: {{ toYaml .Values.controller.admissionWebhooks.patch.tolerations | nindent 8 }}
    {{- end }}
```

Here one of or maybe the major strength of HULL comes into play: setting any object property can be done directly without the need to write any repetitive boilerplate code which goes like: 

> _if this value is provided in `values.yaml` do insert it here 'as is' in the template_

Using HULL all of the shown code blocks can be ignored for our conversion because we can configure the properties without any additional required action at configuration/deployment time if needed. No additional PR needed to make them configurable, they are always accessible if changing them is required.

### The `imagePullSecrets` and `image.registry` feature

A word on `imagePullSecrets` though: it is very easy to configure and use them with HULL. By default, all `registry` objects that are created in HULL will be added as `imagePullSecrets` to the pod specs so images are always pullable even if they are stored on a secured registry. If this behavior is undesired, you can disable this feature simply by setting `createImagePullSecretsFromRegistries` to false in the HULL configuration like this:

```yaml
hull:
  config:
    general:
      createImagePullSecretsFromRegistries: false
```

You can however combine this feature with the possibility to set all pods `registry` fields to a common value. This is allowed for using a fixed registry `server` name (for which the docker registry secret is deployed outside of the Helm chart) or for using the `server` defined in the first found registry secret that is configured in the same chart. 

To provide an example of a real life use case imagine you host all Docker images referenced in your Helm chart in one place - meaning in one Docker registry. Now you may want to switch Docker registry providers - and hence the endpoint - or you need to copy over all Docker images to an airgapped system and host them in a local secured registry such as [Harbor](https://github.com/goharbor/harbor). In both cases you may want to centrally change the registry servers address without having to configure each pod specs `imagePullSecrets` and each containers `image` individually. 

Refer to the [documentation](https://github.com/vidispine/hull/blob/main/hull/README.md#the-config-section) and the following `config` switches:

```yaml
hull:
  config:
    general:
      defaultImageRegistryServer: 
      defaultImageRegistryToFirstRegistrySecretServer:
```

for more details on how to work with this feature.

The `serviceAccountName` configuration was discussed in detail in the tutorial part that dealt with RBAC configuration options so it will not be highlighted again here.

Now to the other aspects of this job:

- the whole object creation is being controlled by having webhooks generally enabled and the patch webhook in particular (`{{ if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhoooks.patch.enabled }}` condition). Those conditions govern creation of all objects in the `/templates/admission-webhooks/job-patch` folder so it makes sense to control the condition in a central place in HULL as well. When discussing the ConfigMaps in the `job-patch` folder you already defined the  `admissionWebhooks.enabled` and `admissionWebhooks.patch.enabled` properties for this reason so you don't need to do it again.

- Helm hooks such as:

  ```yaml
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook- succeeded
  ```

    are simply added to the Job via individual annotations.

- a Capabilities check exists to conditionally include `ttlSecondsAfterFinished` depending on the Kubernetes cluster API version. With HULL being tied to a specific minimum Kubernetes version, specific properties in question are either available or not and the below check is mostly irrelevant:

    ```yaml
    {{- if .Capabilities.APIVersions.Has "batch/v1alpha1" }}
    # Alpha feature since k8s 1.12
    ttlSecondsAfterFinished: 0
    {{- end }}
    ```

- the name suffix will be changed from `admission-create` to `admissioncreate` for simplicity and briefness

- the `args` section requires using a HULL transformation to create an array of arguments that match the dynamic nature of the argument construction. 

Before adding the `admissioncreate` job to your HULL config it makes sense to highlight the differences to the other job in the `templates/admission-webhooks/job-patch` folder, the `job-patchWebhook.yaml`:

```bash
$ cat ../ingress-nginx/templates/admission-webhooks/job-patch/job-patchWebhook.yaml
```

results in this:

```yaml
{{- if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhooks.patch.enabled -}}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "ingress-nginx.fullname" . }}-admission-patch
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": post-install,post-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: admission-webhook
spec:
{{- if .Capabilities.APIVersions.Has "batch/v1alpha1" }}
  # Alpha feature since k8s 1.12
  ttlSecondsAfterFinished: 0
{{- end }}
  template:
    metadata:
      name: {{ include "ingress-nginx.fullname" . }}-admission-patch
    {{- if .Values.controller.admissionWebhooks.patch.podAnnotations }}
      annotations: {{ toYaml .Values.controller.admissionWebhooks.patch.podAnnotations | nindent 8 }}
    {{- end }}
      labels:
        {{- include "ingress-nginx.labels" . | nindent 8 }}
        app.kubernetes.io/component: admission-webhook
    spec:
    {{- if .Values.controller.admissionWebhooks.patch.priorityClassName }}
      priorityClassName: {{ .Values.controller.admissionWebhooks.patch.priorityClassName }}
    {{- end }}
    {{- if .Values.imagePullSecrets }}
      imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
    {{- end }}
      containers:
        - name: patch
          {{- with .Values.controller.admissionWebhooks.patch.image }}
          image: "{{- if .repository -}}{{ .repository }}{{ else }}{{ .registry }}/{{ .image }}{{- end -}}:{{ .tag }}{{- if (.digest) -}} @{{.digest}} {{- end -}}"
          {{- end }}
          imagePullPolicy: {{ .Values.controller.admissionWebhooks.patch.image.pullPolicy }}
          args:
            - patch
            - --webhook-name={{ include "ingress-nginx.fullname" . }}-admission
            - --namespace=$(POD_NAMESPACE)
            - --patch-mutating=false
            - --secret-name={{ include "ingress-nginx.fullname" . }}-admission
            - --patch-failure-policy={{ .Values.controller.admissionWebhooks.failurePolicy }}
          env:
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          {{- if .Values.controller.admissionWebhooks.patchWebhookJob.resources }}
          resources: {{ toYaml .Values.controller.admissionWebhooks.patchWebhookJob.resources | nindent 12 }}
          {{- end }}
      restartPolicy: OnFailure
      serviceAccountName: {{ include "ingress-nginx.fullname" . }}-admission
    {{- if .Values.controller.admissionWebhooks.patch.nodeSelector }}
      nodeSelector: {{ toYaml .Values.controller.admissionWebhooks.patch.nodeSelector | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.admissionWebhooks.patch.tolerations }}
      tolerations: {{ toYaml .Values.controller.admissionWebhooks.patch.tolerations | nindent 8 }}
    {{- end }}
      securityContext:
        runAsNonRoot: true
        runAsUser: {{ .Values.controller.admissionWebhooks.patch.runAsUser }}
{{- end }}
```

If you do a diff now:

```bash
$ diff ../ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml ../ingress-nginx/templates/admission-webhooks/job-patch/job-patchWebhook.yaml
```

the differences are not that many:

```diff
5c5
<   name: {{ include "ingress-nginx.fullname" . }}-admission-create
---
>   name: {{ include "ingress-nginx.fullname" . }}-admission-patch
8c8
<     "helm.sh/hook": pre-install,pre-upgrade
---
>     "helm.sh/hook": post-install,post-upgrade
20c20
<       name: {{ include "ingress-nginx.fullname" . }}-admission-create
---
>       name: {{ include "ingress-nginx.fullname" . }}-admission-patch
35c35
<         - name: create
---
>         - name: patch
41,42c41,42
<             - create
<             - --host={{ include "ingress-nginx.controller.fullname" . }}-admission,{{ include "ingress-nginx.controller.fullname" . }}-admission.$(POD_NAMESPACE).svc
---
>             - patch
>             - --webhook-name={{ include "ingress-nginx.fullname" . }}-admission
43a44
>             - --patch-mutating=false
44a46
>             - --patch-failure-policy={{ .Values.controller.admissionWebhooks.failurePolicy }}
50,51c52,53
<           {{- if .Values.controller.admissionWebhooks.createSecretJob.resources }}
<           resources: {{ toYaml .Values.controller.admissionWebhooks.createSecretJob.resources | nindent 12 }}
---
>           {{- if .Values.controller.admissionWebhooks.patchWebhookJob.resources }}
>           resources: {{ toYaml .Values.controller.admissionWebhooks.patchWebhookJob.resources | nindent 12 }}
```

Having only slight differences between the only two jobs enables us to utilize some shared defaults for them via the `_HULL_OBJECT_TYPE_DEFAULT_` feature. With this special object key you can define properties that are applied to all objects of the parents type. This saves some typing and repetition as you will see. 

> Note that these default properties cannot themselves be derived via a `_HULL_TRANSFORMATION`, they need to be static values.

Add a new global property `failurePolicy` first to `hull.config.specific` which is referenced in one of the Job's `args` and later in a webhook so it should be stored in a central place:

```bash
$ sed '/^\s\s\s\s\s\s\s\sadmissionWebhooks:/r'<(
      echo "          failurePolicy: Fail"
    ) -i -- values.yaml
```

and then create the two Jobs like this:

```bash
$ echo '  objects:
    job:
      _HULL_OBJECT_TYPE_DEFAULT_:
        annotations:
          "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
        ttlSecondsAfterFinished: 0
        pod:
          containers:
            _HULL_OBJECT_TYPE_DEFAULT_:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/kube-webhook-certgen
                tag: v1.1.1
              imagePullPolicy: IfNotPresent
              env:
                POD_NAMESPACE:
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
          restartPolicy: OnFailure
          nodeSelector:
            kubernetes.io/os: linux
          securityContext:
            runAsNonRoot: true
            runAsUser: 2000      
      admissioncreate:
        enabled: _HT?and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled
        annotations:
          "helm.sh/hook": pre-install,pre-upgrade
        pod:
          containers:
            create:
              args: _HT![
                      "create",
                      "--host={{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "admission") }},{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "admission") }}.$(POD_NAMESPACE).svc",
                      "--namespace=$(POD_NAMESPACE)",
                      "--secret-name={{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "admission") }}"
                    ]
      admissionpatch:
        enabled: _HT?and (index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled (index . "$").Values.hull.config.specific.controller.admissionWebhooks.patch.enabled
        annotations:
          "helm.sh/hook": post-install,post-upgrade
        pod:
          containers:
            patch:
              args: _HT![
                      "patch",
                      "--webhook-name={{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "admission") }},{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "admission") }}.$(POD_NAMESPACE).svc",
                      "--namespace=$(POD_NAMESPACE)",
                      "--patch-mutating=false",
                      "--secret-name={{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "admission") }}",
                      "--patch-failure-policy={{ (index . "$").Values.hull.config.specific.controller.admissionWebhooks.failurePolicy }}"
                    ]' >> values.yaml
``` 

A quick check if everything works and looks as expected:

```bash
$ helm template .
```

shows the `default` RBAC objects and ServiceAccount and the two Jobs:

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
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admissioncreate
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admissioncreate
spec:
  template:
    metadata:
      annotations:
        helm.sh/hook: pre-install,pre-upgrade
        helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
      labels:
        app.kubernetes.io/component: admissioncreate
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - args:
        - create
        - --host=release-name-ingress-nginx-hull-admission,release-name-ingress-nginx-hull-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=release-name-ingress-nginx-hull-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: create
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
  ttlSecondsAfterFinished: 0
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admissionpatch
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admissionpatch
spec:
  template:
    metadata:
      annotations:
        helm.sh/hook: post-install,post-upgrade
        helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
      labels:
        app.kubernetes.io/component: admissionpatch
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=release-name-ingress-nginx-hull-admission,release-name-ingress-nginx-hull-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=release-name-ingress-nginx-hull-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: patch
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
  ttlSecondsAfterFinished: 0
```

### DaemonSets in HULL

Progressing to create the remaining two Deployments and the DaemonSet, take a look at the `controller` DaemonSet first:

```bash
$ cat ../ingress-nginx/templates/controller-daemonset.yaml
```

which reveals the pretty long-ish definition of it:

```yaml
{{- if or (eq .Values.controller.kind "DaemonSet") (eq .Values.controller.kind "Both") -}}
{{- include  "isControllerTagValid" . -}}
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
    {{- with .Values.controller.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
  name: {{ include "ingress-nginx.controller.fullname" . }}
  namespace: {{ .Release.Namespace }}
  {{- if .Values.controller.annotations }}
  annotations: {{ toYaml .Values.controller.annotations | nindent 4 }}
  {{- end }}
spec:
  selector:
    matchLabels:
      {{- include "ingress-nginx.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: controller
  revisionHistoryLimit: {{ .Values.revisionHistoryLimit }}
  {{- if .Values.controller.updateStrategy }}
  updateStrategy: {{ toYaml .Values.controller.updateStrategy | nindent 4 }}
  {{- end }}
  minReadySeconds: {{ .Values.controller.minReadySeconds }}
  template:
    metadata:
    {{- if .Values.controller.podAnnotations }}
      annotations:
      {{- range $key, $value := .Values.controller.podAnnotations }}
        {{ $key }}: {{ $value | quote }}
      {{- end }}
    {{- end }}
      labels:
        {{- include "ingress-nginx.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: controller
      {{- if .Values.controller.podLabels }}
        {{- toYaml .Values.controller.podLabels | nindent 8 }}
      {{- end }}
    spec:
    {{- if .Values.controller.dnsConfig }}
      dnsConfig: {{ toYaml .Values.controller.dnsConfig | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.hostname }}
      hostname: {{ toYaml .Values.controller.hostname | nindent 8 }}
    {{- end }}
      dnsPolicy: {{ .Values.controller.dnsPolicy }}
    {{- if .Values.imagePullSecrets }}
      imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.priorityClassName }}
      priorityClassName: {{ .Values.controller.priorityClassName }}
    {{- end }}
    {{- if or .Values.controller.podSecurityContext .Values.controller.sysctls }}
      securityContext:
    {{- end }}
    {{- if .Values.controller.podSecurityContext  }}
        {{- toYaml .Values.controller.podSecurityContext | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.sysctls }}
        sysctls:
    {{- range $sysctl, $value := .Values.controller.sysctls }}
        - name: {{ $sysctl | quote }}
          value: {{ $value | quote }}
    {{- end }}
    {{- end }}
      containers:
        - name: {{ .Values.controller.containerName }}
          {{- with .Values.controller.image }}
          image: "{{- if .repository -}}{{ .repository }}{{ else }}{{ .registry }}/{{ .image }}{{- end -}}:{{ .tag }}{{- if (.digest) -}} @{{.digest}} {{- end -}}"
          {{- end }}
          imagePullPolicy: {{ .Values.controller.image.pullPolicy }}
        {{- if .Values.controller.lifecycle }}
          lifecycle: {{ toYaml .Values.controller.lifecycle | nindent 12 }}
        {{- end }}
          args:
            - /nginx-ingress-controller
          {{- if .Values.defaultBackend.enabled }}
            - --default-backend-service=$(POD_NAMESPACE)/{{ include "ingress-nginx.defaultBackend.fullname" . }}
          {{- end }}
          {{- if .Values.controller.publishService.enabled }}
            - --publish-service={{ template "ingress-nginx.controller.publishServicePath" . }}
          {{- end }}
            - --election-id={{ .Values.controller.electionID }}
            - --controller-class={{ .Values.controller.ingressClassResource.controllerValue }}
            - --configmap={{ default "$(POD_NAMESPACE)" .Values.controller.configMapNamespace }}/{{ include "ingress-nginx.controller.fullname" . }}
          {{- if .Values.tcp }}
            - --tcp-services-configmap={{ default "$(POD_NAMESPACE)" .Values.controller.tcp.configMapNamespace }}/{{ include "ingress-nginx.fullname" . }}-tcp
          {{- end }}
          {{- if .Values.udp }}
            - --udp-services-configmap={{ default "$(POD_NAMESPACE)" .Values.controller.udp.configMapNamespace }}/{{ include "ingress-nginx.fullname" . }}-udp
          {{- end }}
          {{- if .Values.controller.scope.enabled }}
            - --watch-namespace={{ default "$(POD_NAMESPACE)" .Values.controller.scope.namespace }}
          {{- end }}
          {{- if and .Values.controller.reportNodeInternalIp .Values.controller.hostNetwork }}
            - --report-node-internal-ip-address={{ .Values.controller.reportNodeInternalIp }}
          {{- end }}
          {{- if .Values.controller.admissionWebhooks.enabled }}
            - --validating-webhook=:{{ .Values.controller.admissionWebhooks.port }}
            - --validating-webhook-certificate={{ .Values.controller.admissionWebhooks.certificate }}
            - --validating-webhook-key={{ .Values.controller.admissionWebhooks.key }}
          {{- end }}
          {{- if .Values.controller.maxmindMirror }}
            - --maxmind-mirror={{ .Values.controller.maxmindMirror }}
          {{- end}}
          {{- if .Values.controller.maxmindLicenseKey }}
            - --maxmind-license-key={{ .Values.controller.maxmindLicenseKey }}
          {{- end }}
          {{- if not (eq .Values.controller.healthCheckPath "/healthz") }}
            - --health-check-path={{ .Values.controller.healthCheckPath }}
          {{- end }}
          {{- if .Values.controller.healthCheckHost }}
            - --healthz-host={{ .Values.controller.healthCheckHost }}
          {{- end }}
          {{- if .Values.controller.ingressClassByName }}
            - --ingress-class-by-name=true
          {{- end }}
          {{- if .Values.controller.watchIngressWithoutClass }}
            - --watch-ingress-without-class=true
          {{- end }}
          {{- range $key, $value := .Values.controller.extraArgs }}
            {{- /* Accept keys without values or with false as value */}}
            {{- if eq ($value | quote | len) 2 }}
            - --{{ $key }}
            {{- else }}
            - --{{ $key }}={{ $value }}
            {{- end }}
          {{- end }}
          securityContext:
            capabilities:
                drop:
                - ALL
                add:
                - NET_BIND_SERVICE
            runAsUser: {{ .Values.controller.image.runAsUser }}
            allowPrivilegeEscalation: {{ .Values.controller.image.allowPrivilegeEscalation }}
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          {{- if .Values.controller.enableMimalloc }}
            - name: LD_PRELOAD
              value: /usr/local/lib/libmimalloc.so
          {{- end }}
          {{- if .Values.controller.extraEnvs }}
            {{- toYaml .Values.controller.extraEnvs | nindent 12 }}
          {{- end }}
          {{- if .Values.controller.startupProbe }}
          startupProbe: {{ toYaml .Values.controller.startupProbe | nindent 12 }}
          {{- end }}
          livenessProbe: {{ toYaml .Values.controller.livenessProbe | nindent 12 }}
          readinessProbe: {{ toYaml .Values.controller.readinessProbe | nindent 12 }}
          ports:
          {{- range $key, $value := .Values.controller.containerPort }}
            - name: {{ $key }}
              containerPort: {{ $value }}
              protocol: TCP
              {{- if $.Values.controller.hostPort.enabled }}
              hostPort: {{ index $.Values.controller.hostPort.ports $key | default $value }}
              {{- end }}
          {{- end }}
          {{- if .Values.controller.metrics.enabled }}
            - name: metrics
              containerPort: {{ .Values.controller.metrics.port }}
              protocol: TCP
          {{- end }}
          {{- if .Values.controller.admissionWebhooks.enabled }}
            - name: webhook
              containerPort: {{ .Values.controller.admissionWebhooks.port }}
              protocol: TCP
          {{- end }}
          {{- range $key, $value := .Values.tcp }}
            - name: {{ $key }}-tcp
              containerPort: {{ $key }}
              protocol: TCP
              {{- if $.Values.controller.hostPort.enabled }}
              hostPort: {{ $key }}
              {{- end }}
          {{- end }}
          {{- range $key, $value := .Values.udp }}
            - name: {{ $key }}-udp
              containerPort: {{ $key }}
              protocol: UDP
              {{- if $.Values.controller.hostPort.enabled }}
              hostPort: {{ $key }}
              {{- end }}
          {{- end }}
        {{- if (or .Values.controller.customTemplate.configMapName .Values.controller.extraVolumeMounts .Values.controller.admissionWebhooks.enabled) }}
          volumeMounts:
          {{- if .Values.controller.customTemplate.configMapName }}
            - mountPath: /etc/nginx/template
              name: nginx-template-volume
              readOnly: true
          {{- end }}
          {{- if .Values.controller.admissionWebhooks.enabled }}
            - name: webhook-cert
              mountPath: /usr/local/certificates/
              readOnly: true
          {{- end }}
          {{- if .Values.controller.extraVolumeMounts }}
            {{- toYaml .Values.controller.extraVolumeMounts | nindent 12 }}
          {{- end }}
        {{- end }}
        {{- if .Values.controller.resources }}
          resources: {{ toYaml .Values.controller.resources | nindent 12 }}
        {{- end }}
      {{- if .Values.controller.extraContainers }}
        {{ toYaml .Values.controller.extraContainers | nindent 8 }}
      {{- end }}
    {{- if .Values.controller.extraInitContainers }}
      initContainers: {{ toYaml .Values.controller.extraInitContainers | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.hostNetwork }}
      hostNetwork: {{ .Values.controller.hostNetwork }}
    {{- end }}
    {{- if .Values.controller.nodeSelector }}
      nodeSelector: {{ toYaml .Values.controller.nodeSelector | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.tolerations }}
      tolerations: {{ toYaml .Values.controller.tolerations | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.affinity }}
      affinity: {{ toYaml .Values.controller.affinity | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.topologySpreadConstraints }}
      topologySpreadConstraints: {{ toYaml .Values.controller.topologySpreadConstraints | nindent 8 }}
    {{- end }}
      serviceAccountName: {{ template "ingress-nginx.serviceAccountName" . }}
      terminationGracePeriodSeconds: {{ .Values.controller.terminationGracePeriodSeconds }}
    {{- if (or .Values.controller.customTemplate.configMapName .Values.controller.extraVolumeMounts .Values.controller.admissionWebhooks.enabled .Values.controller.extraVolumes) }}
      volumes:
      {{- if .Values.controller.customTemplate.configMapName }}
        - name: nginx-template-volume
          configMap:
            name: {{ .Values.controller.customTemplate.configMapName }}
            items:
            - key: {{ .Values.controller.customTemplate.configMapKey }}
              path: nginx.tmpl
      {{- end }}
      {{- if .Values.controller.admissionWebhooks.enabled }}
        - name: webhook-cert
          secret:
            secretName: {{ include "ingress-nginx.fullname" . }}-admission
      {{- end }}
      {{- if .Values.controller.extraVolumes }}
        {{ toYaml .Values.controller.extraVolumes | nindent 8 }}
      {{- end }}
    {{- end }}
{{- end }}
```

Looking closer you'll notice actually the bulk of the lines are those 1:1 projections that you need not define when writing with HULL. But before getting to work, again a review of interesting aspects of this DaemonSet:

-  From the first line 

    ```yaml
    {{- if or (eq .Values.controller.kind "DaemonSet") (eq .Values.controller.kind "Both") -}}
    ``` 

    you can deduce that you should be able to deploy the only the `controller` Deployment, only the `controller` DaemonSet or both. All cases are simply managed in HULL by setting the corresponding `enabled` properties to `true` or `false`-. To reflect the default setting (which is `.Values.controller.kind: "Deployment"` so Deployment only) you'll enable the Deployment and disable the DaemonSet by default. 

-  Second line `{{- include  "isControllerTagValid" . -}}` implements an internal check for a minimum version which is defined in `_helpers.tpl`:

    ```bash
    $ cat ../ingress-nginx/templates/_helpers.tpl | grep 'isControllerTagValid' -B 10 -A 10
    ```

    ```yaml
    {{- if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
    {- print "policy" -}}
    {{- else -}}
    {{- print "extensions" -}}
    {{- end -}}
    {{- end -}}

    {{/*
    Check the ingress controller version tag is at most three versions behind the last release
    */}}
    {{- define "isControllerTagValid" -}}
    {{- if not (semverCompare ">=0.27.0-0" 
    .Values.controller.image.tag) -}}
    {{- fail "Controller container image tag should be 0.27.0 or 
    higher" -}}
    {{- end -}}
    {{- end -}}

    {{/*
    IngressClass parameters.
    */}}
    {{- define "ingressClass.parameters" -}}
      {{- if .Values.controller.ingressClassResource.parameters 
    -}}
    ```

    The Helm `fail` functionality is not supported but alternatively here you could make sure that a configured `tag` version is raised to `0.25` if it is found to be below that. 

- The actual main container name can be changed in the original chart:

    ```yaml
        containers:
        - name: {{ .Values.controller.containerName }}
    ```

    Since with HULL the name is derived by the container's key, changing the key in a system specific configuration would require to redefine all the containers content which is not useful. Hence you have to stick with the static defined container name `controller` in the HULL version of the chart.

- A lot of the pod specific properties are referencing to properties under the `controller` section in the original charts `values.yaml`. Almost all of them are also referenced in the `controller` Deployment as you will see later. However upon closer inspection these properties fall into two classes:

    1. concrete properties which are diretly associated with an object property (e.g. `imagePullPolicy`, `affinity`, `tolerations`, ...)
    2. abstract properties that are not directly associated with an object property but are used to derive them.

    The shared abstract properties need to be modeled somewhere under `hull.config.specific` but for the concrete ones you need to decide whether you lean more towards transparency or towards abstraction. 

    - if you aim for transparency it is best to define the properties directly on the `controller` DaemonSet and Deployment objects so the same property can have different values configured for both of them. 
    
      _Advantage here is that it is more straightforward to understand where each object's property value is defined._
      
      _Disadvantage is that this weakens the likely intention to have the DaemonSet and Deployment as alike as possible since it allows to involuntarily have unwanted differences between them._

    - if you aim for abstraction you should define all the properties that are being referenced from both Deplyoment and DaemonSet under the `hull.config.specific` section and reference the central values. 
    
      _Disadvantage is that it obfuscates where a properties value comes from more and requires much more effort to set up._
    
      _Advantages are that the intent of having `controller` DaemonSet and Deployment as much alike as possible is better fulfillable and it would still be possible to overwrite any individual property if wanted._ 

    Here for simplicity and transparency you should go with omitting the properties that are not required on the objects so that they can be changed individually.

Ok so first insert the new `hull.config.specific` properties so they can be referenced to:

```bash
$ sed '/^\s\s\s\s\s\scontroller:/r'<(
      echo "        publishService:"
      echo "          enabled: true"
      echo "          pathOverride: \"\""
      echo "        ingressClassResource:"
      echo "          controllerValue: \"k8s.io/ingress-nginx\""
      echo "          name: nginx"
      echo "          enabled: true"
      echo "          default: false"
      echo "        reportNodeInternalIp: false"
      echo "        hostNetwork: false"
      echo "        maxMindMirror: "
      echo "        maxmindLicenseKey: \"\""
      echo "        healthCheckHost: \"\""
      echo "        healthCheckPath: \"/healthz\""
      echo "        ingressClassByName: false"
      echo "        watchIngressWithoutClass: false"
      echo "        enableMimalloc: true"
      echo "        metrics:"
      echo "          enabled: false"
      echo "        customTemplate:"
      echo "          configMapName: \"\""
      echo "          configMapKey: \"\""
     ) -i -- values.yaml && 
     sed '/^\s\s\s\s\s\s\s\s\s\stcp:/r'<(
      echo "            namespace: \"\""
     ) -i -- values.yaml && 
     sed '/^\s\s\s\s\s\s\s\s\s\sudp:/r'<(
      echo "            namespace: \"\""
     ) -i -- values.yaml && 
     sed '/^\s\s\s\s\s\s\s\sconfigMaps:/r'<(
      echo "          controller:"
      echo "            namespace: \"\""
     ) -i -- values.yaml && 
     sed '/^\s\s\s\s\s\s\s\sadmissionWebhooks:/r'<(
      echo "          port: 8443"
      echo "          certificate: \"/usr/local/certificates/cert\""
      echo "          key: \"/usr/local/certificates/key\""
     ) -i -- values.yaml
```

This is how you can compress the `controller` DaemonSet's definition using HULL:

```bash
$ echo '    daemonset:
      controller:
        enabled: false
        revisionHistoryLimit: 10
        minReadySeconds: 0 
        pod:
          dnsPolicy: ClusterFirst
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
              imagePullPolicy: IfNotPresent
              lifecycle:
                preStop:
                  exec:
                    command:
                      - /wait-shutdown
              args: _HT![
                  {{- $p := (index . "$") }}
                  
                  "-/nginx-ingress-controller",
                  
                  {{- if $p.Values.hull.config.specific.defaultBackend.enabled }}
                  "--default-backend-service=$(POD_NAMESPACE)/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "defaultbackend") }}",
                  {{- end }}
                  
                  {{- with $p.Values.hull.config.specific.controller }}
                  {{- if .publishService.enabled }}
                  "--publish-service={{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "controller")) .publishService.pathOverride }}",
                  {{- end }}
                  
                  "--election-id={{ .electionID }}",
                  
                  "--controller-class={{ .ingressClassResource.controllerValue }}",
                  
                  "--configmap={{ default "$(POD_NAMESPACE)" .configMaps.controller.namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "controller") }}",
                  
                  {{- if .configMaps.tcp.mappings }}
                  "--tcp-services-configmap={{ default "$(POD_NAMESPACE)" .configMaps.tcp.namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "tcp") }}",
                  {{ end }}
                  
                  {{- if .configMaps.udp.mappings }}
                  "--udp-services-configmap={{ default "$(POD_NAMESPACE)" .configMaps.udp.namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "udp") }}",
                  {{ end }}
                  
                  {{- if .scope.enabled }}
                  "--watch-namespace={{ default "$(POD_NAMESPACE)" .scope.namespace }}",
                  {{- end }}
                  
                  {{- if and .reportNodeInternalIp .hostNetwork }}
                  "--report-node-internal-ip-address={{ .reportNodeInternalIp }}",
                  {{- end }}
                  
                  {{- if .admissionWebhooks.enabled }}
                  "--validating-webhook={{ .admissionWebhooks.port }}",
                  "--validating-webhook-certificate={{ .admissionWebhooks.certificate }}",
                  "--validating-webhook-key={{ .admissionWebhooks.key }}",
                  {{- end }}
                  
                  {{- if .maxMindMirror }}
                  "--healthz-host={{ .healthCheckHost }}",
                  {{- end }}
                  
                  {{- if .maxmindLicenseKey }}
                  "--maxmind-license-key={{ .maxmindLicenseKey }}",
                  {{- end }}
                  
                  {{- if .healthCheckHost }}
                  "--healthz-host={{ .healthCheckHost }}",
                  {{- end }}                    
                  {{- if not (eq .healthCheckPath "/healthz") }}
                  "--health-check-path={{ .Values.controller.healthCheckPath }}",
                  {{- end }}
                  {{- if .ingressClassByName }}
                  "--ingress-class-by-name=true",
                  {{- end }}
                  
                  {{- if .watchIngressWithoutClass }}
                  "--watch-ingress-without-class=true",
                  {{- end }}
                  {{- end }}
                ]
              securityContext:
                capabilities:
                    drop:
                    - ALL
                    add:
                    - NET_BIND_SERVICE
                runAsUser: 101
                allowPrivilegeEscalation: true
              env:
                POD_NAME:
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.name
                POD_NAMESPACE:
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
                LD_PRELOAD:
                  enabled: _HT?(index . "$").Values.hull.config.specific.controller.enableMimalloc
                  value: /usr/local/lib/libmimalloc.so
              livenessProbe: 
                httpGet:
                  # should match container.healthCheckPath
                  path: "/healthz"
                  port: 10254
                  scheme: HTTP
                initialDelaySeconds: 10
                periodSeconds: 10
                timeoutSeconds: 1
                successThreshold: 1
                failureThreshold: 5
              readinessProbe:
                httpGet:
                  # should match container.healthCheckPath
                  path: "/healthz"
                  port: 10254
                  scheme: HTTP
                initialDelaySeconds: 10
                periodSeconds: 10
                timeoutSeconds: 1
                successThreshold: 1
                failureThreshold: 3
              
              ports: |-
                    _HT!{
                      {{- $p := (index . "$") }}
                      {{ if $p.Values.hull.config.specific.controller.metrics.enabled }}
                      metrics: { containerPort: 10254, protocol: TCP },
                      {{ end }}
                      
                      {{ if $p.Values.hull.config.specific.controller.admissionWebhooks.enabled }}
                      webhook: { containerPort: 8443, protocol: TCP },
                      {{ end }}
                      
                      {{ range $a,$b := $p.Values.hull.config.specific.controller.configMaps.tcp.mappings }}
                      {{ $a }}-tcp: { containerPort: {{ printf "%s" $a }} },
                      {{ end }}
                      
                      {{ range $a,$b := $p.Values.hull.config.specific.controller.configMaps.udp.mappings }}
                      {{ $a }}-udp: { containerPort: {{ printf "%s" $a }} },
                      {{ end }}
                      
                      http: { containerPort: 80, protocol: TCP },
                      
                      https: { containerPort: 443, protocol: TCP }
                    }
              volumeMounts:
                customTemplate:
                  enabled: _HT?(index . "$").Values.hull.config.specific.controller.customTemplate.configMapName
                  mountPath: /etc/nginx/template
                  name: nginx-template-volume
                  readOnly: true
                webhook:
                  enabled: _HT?(index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled
                  name: webhook-cert
                  mountPath: /usr/local/certificates/
                  readOnly: true
              resources:
                requests:
                  cpu: 100m
                  memory: 90Mi
          hostNetwork: false
          terminationGracePeriodSeconds: 300
          volumes:
            'nginx-template-volume':
              enabled: _HT?(index . "$").Values.hull.config.specific.controller.customTemplate.configMapName
              configMap:
                name: _HT*hull.config.specific.controller.customTemplate.configMapName
                items:
                - key: _HT*hull.config.specific.controller.customTemplate.configMapKey
                  path: nginx.tmpl
            'webhook-cert':
              enabled: _HT?(index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled
              secret:
                secretName: admission' >> values.yaml
```

You can enable it with this overlay:

```bash
$ echo 'hull:
  objects:
    daemonset:
      controller: 
        enabled: true' > ../configs/enable-daemonset.yaml
```

and with:

```bash
$ helm template -f ../configs/enable-daemonset.yaml .
```
view how it is rendered:

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
kind: DaemonSet
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
  minReadySeconds: 0
  revisionHistoryLimit: 10
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
      - args:
        - -/nginx-ingress-controller
        - --publish-service=release-name-ingress-nginx-hull-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --configmap=$(POD_NAMESPACE)/release-name-ingress-nginx-hull-controller
        - --validating-webhook=8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      dnsPolicy: ClusterFirst
      hostNetwork: false
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: release-name-ingress-nginx-hull-default
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: release-name-ingress-nginx-hull-admission
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admissioncreate
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admissioncreate
spec:
  template:
    metadata:
      annotations:
        helm.sh/hook: pre-install,pre-upgrade
        helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
      labels:
        app.kubernetes.io/component: admissioncreate
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - args:
        - create
        - --host=release-name-ingress-nginx-hull-admission,release-name-ingress-nginx-hull-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=release-name-ingress-nginx-hull-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: create
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
  ttlSecondsAfterFinished: 0
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admissionpatch
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admissionpatch
spec:
  template:
    metadata:
      annotations:
        helm.sh/hook: post-install,post-upgrade
        helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
      labels:
        app.kubernetes.io/component: admissionpatch
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=release-name-ingress-nginx-hull-admission,release-name-ingress-nginx-hull-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=release-name-ingress-nginx-hull-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: patch
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
  ttlSecondsAfterFinished: 0
```

The Deployment specification looks virtually identical to the DaemonSet when highlighting the differences:

```bash
$ diff ../ingress-nginx/templates/controller-daemonset.yaml ../ingress-nginx/templates/controller-deployment.yaml
```

giving:

```diff
1c1
< {{- if or (eq .Values.controller.kind "DaemonSet") (eq .Values.controller.kind "Both") -}}
---
> {{- if or (eq .Values.controller.kind "Deployment") (eq .Values.controller.kind "Both") -}}
4c4
< kind: DaemonSet
---
> kind: Deployment
21a22,24
>   {{- if not .Values.controller.autoscaling.enabled }}
>   replicas: {{ .Values.controller.replicaCount }}
>   {{- end }}
24c27,28
<   updateStrategy: {{ toYaml .Values.controller.updateStrategy | nindent 4 }}
---
>   strategy:
>     {{ toYaml .Values.controller.updateStrategy | nindent 4 }}
58c62
<     {{- if .Values.controller.podSecurityContext  }}
---
>     {{- if .Values.controller.podSecurityContext }}
105,107d108
<           {{- if .Values.controller.maxmindMirror }}
<             - --maxmind-mirror={{ .Values.controller.maxmindMirror }}
<           {{- end}}
111,113d111
<           {{- if not (eq .Values.controller.healthCheckPath "/healthz") }}
<             - --health-check-path={{ .Values.controller.healthCheckPath }}
<           {{- end }}
115a114,116
>           {{- end }}
>           {{- if not (eq .Values.controller.healthCheckPath "/healthz") }}
>             - --health-check-path={{ .Values.controller.healthCheckPath }}
``` 

so it is easy to add it into the `values.yaml` with:

```bash
$ echo '    deployment:
      controller:
        enabled: true
        revisionHistoryLimit: 10
        minReadySeconds: 0 
        pod:
          dnsPolicy: ClusterFirst
          containers:
            controller:
              image:
                registry: k8s.gcr.io
                repository: ingress-nginx/controller
                tag: "v1.0.4"
              imagePullPolicy: IfNotPresent
              lifecycle:
                preStop:
                  exec:
                    command:
                      - /wait-shutdown
              args: _HT![
                  {{- $p := (index . "$") }}
                  
                  "-/nginx-ingress-controller",
                  
                  {{- if $p.Values.hull.config.specific.defaultBackend.enabled }}
                  "--default-backend-service=$(POD_NAMESPACE)/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "defaultbackend") }}",
                  {{- end }}
                  
                  {{- with $p.Values.hull.config.specific.controller }}
                  {{- if .publishService.enabled }}
                  "--publish-service={{ default (include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "controller")) .publishService.pathOverride }}",
                  {{- end }}
                  
                  "--election-id={{ .electionID }}",
                  
                  "--controller-class={{ .ingressClassResource.controllerValue }}",
                  
                  "--configmap={{ default "$(POD_NAMESPACE)" .configMaps.controller.namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "controller") }}",
                  
                  {{- if .configMaps.tcp.mappings }}
                  "--tcp-services-configmap={{ default "$(POD_NAMESPACE)" .configMaps.tcp.namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "tcp") }}",
                  {{ end }}
                  
                  {{- if .configMaps.udp.mappings }}
                  "--udp-services-configmap={{ default "$(POD_NAMESPACE)" .configMaps.udp.namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $p "COMPONENT" "udp") }}",
                  {{ end }}
                  
                  {{- if .scope.enabled }}
                  "--watch-namespace={{ default "$(POD_NAMESPACE)" .scope.namespace }}",
                  {{- end }}
                  
                  {{- if and .reportNodeInternalIp .hostNetwork }}
                  "--report-node-internal-ip-address={{ .reportNodeInternalIp }}",
                  {{- end }}
                  
                  {{- if .admissionWebhooks.enabled }}
                  "--validating-webhook={{ .admissionWebhooks.port }}",
                  "--validating-webhook-certificate={{ .admissionWebhooks.certificate }}",
                  "--validating-webhook-key={{ .admissionWebhooks.key }}",
                  {{- end }}
                  
                  {{- if .maxmindLicenseKey }}
                  "--maxmind-license-key={{ .maxmindLicenseKey }}",
                  {{- end }}
                  
                  {{- if .healthCheckHost }}
                  "--healthz-host={{ .healthCheckHost }}",
                  {{- end }}                    
                  {{- if not (eq .healthCheckPath "/healthz") }}
                  "--health-check-path={{ .Values.controller.healthCheckPath }}",
                  {{- end }}
                  {{- if .ingressClassByName }}
                  "--ingress-class-by-name=true",
                  {{- end }}
                  
                  {{- if .watchIngressWithoutClass }}
                  "--watch-ingress-without-class=true",
                  {{- end }}
                  {{- end }}
                ]
              securityContext:
                capabilities:
                    drop:
                    - ALL
                    add:
                    - NET_BIND_SERVICE
                runAsUser: 101
                allowPrivilegeEscalation: true
              env:
                POD_NAME:
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.name
                POD_NAMESPACE:
                  valueFrom:
                    fieldRef:
                      fieldPath: metadata.namespace
                LD_PRELOAD:
                  enabled: _HT?(index . "$").Values.hull.config.specific.controller.enableMimalloc
                  value: /usr/local/lib/libmimalloc.so
              livenessProbe: 
                httpGet:
                  # should match container.healthCheckPath
                  path: "/healthz"
                  port: 10254
                  scheme: HTTP
                initialDelaySeconds: 10
                periodSeconds: 10
                timeoutSeconds: 1
                successThreshold: 1
                failureThreshold: 5
              readinessProbe:
                httpGet:
                  # should match container.healthCheckPath
                  path: "/healthz"
                  port: 10254
                  scheme: HTTP
                initialDelaySeconds: 10
                periodSeconds: 10
                timeoutSeconds: 1
                successThreshold: 1
                failureThreshold: 3
              
              ports: |-
                    _HT!{
                      {{- $p := (index . "$") }}
                      {{ if $p.Values.hull.config.specific.controller.metrics.enabled }}
                      metrics: { containerPort: 10254, protocol: TCP },
                      {{ end }}
                      
                      {{ if $p.Values.hull.config.specific.controller.admissionWebhooks.enabled }}
                      webhook: { containerPort: 8443, protocol: TCP },
                      {{ end }}
                      
                      {{ range $a,$b := $p.Values.hull.config.specific.controller.configMaps.tcp.mappings }}
                      {{ $a }}-tcp: { containerPort: {{ printf "%s" $a }} },
                      {{ end }}
                      
                      {{ range $a,$b := $p.Values.hull.config.specific.controller.configMaps.udp.mappings }}
                      {{ $a }}-udp: { containerPort: {{ printf "%s" $a }} },
                      {{ end }}
                      
                      http: { containerPort: 80, protocol: TCP },
                      
                      https: { containerPort: 443, protocol: TCP }
                    }
              volumeMounts:
                customTemplate:
                  enabled: _HT?(index . "$").Values.hull.config.specific.controller.customTemplate.configMapName
                  mountPath: /etc/nginx/template
                  name: nginx-template-volume
                  readOnly: true
                webhook:
                  enabled: _HT?(index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled
                  name: webhook-cert
                  mountPath: /usr/local/certificates/
                  readOnly: true
              resources:
                requests:
                  cpu: 100m
                  memory: 90Mi
          hostNetwork: false
          terminationGracePeriodSeconds: 300
          volumes:
            'nginx-template-volume':
              enabled: _HT?(index . "$").Values.hull.config.specific.controller.customTemplate.configMapName
              configMap:
                name: _HT*hull.config.specific.controller.customTemplate.configMapName
                items:
                - key: _HT*hull.config.specific.controller.customTemplate.configMapKey
                  path: nginx.tmpl
            'webhook-cert':
              enabled: _HT?(index . "$").Values.hull.config.specific.controller.admissionWebhooks.enabled
              secret:
                secretName: admission' >> values.yaml
```

and get a new `values.yaml`. When you now render it without overlay file the Deployment will be created (since it is enabled by default contrary to the DaemonSet):

```bash
$ helm template .
```

renders as:

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
  minReadySeconds: 0
  revisionHistoryLimit: 10
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
      - args:
        - -/nginx-ingress-controller
        - --publish-service=release-name-ingress-nginx-hull-controller
        - --election-id=ingress-controller-leader
        - --controller-class=k8s.io/ingress-nginx
        - --configmap=$(POD_NAMESPACE)/release-name-ingress-nginx-hull-controller
        - --validating-webhook=8443
        - --validating-webhook-certificate=/usr/local/certificates/cert
        - --validating-webhook-key=/usr/local/certificates/key
        env:
        - name: LD_PRELOAD
          value: /usr/local/lib/libmimalloc.so
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/controller:v1.0.4
        imagePullPolicy: IfNotPresent
        lifecycle:
          preStop:
            exec:
              command:
              - /wait-shutdown
        livenessProbe:
          failureThreshold: 5
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        name: controller
        ports:
        - containerPort: 80
          name: http
          protocol: TCP
        - containerPort: 443
          name: https
          protocol: TCP
        - containerPort: 8443
          name: webhook
          protocol: TCP
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          requests:
            cpu: 100m
            memory: 90Mi
        securityContext:
          allowPrivilegeEscalation: true
          capabilities:
            add:
            - NET_BIND_SERVICE
            drop:
            - ALL
          runAsUser: 101
        volumeMounts:
        - mountPath: /usr/local/certificates/
          name: webhook-cert
          readOnly: true
      dnsPolicy: ClusterFirst
      hostNetwork: false
      imagePullSecrets: []
      initContainers: []
      serviceAccountName: release-name-ingress-nginx-hull-default
      terminationGracePeriodSeconds: 300
      volumes:
      - name: webhook-cert
        secret:
          secretName: release-name-ingress-nginx-hull-admission
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: pre-install,pre-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admissioncreate
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admissioncreate
spec:
  template:
    metadata:
      annotations:
        helm.sh/hook: pre-install,pre-upgrade
        helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
      labels:
        app.kubernetes.io/component: admissioncreate
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - args:
        - create
        - --host=release-name-ingress-nginx-hull-admission,release-name-ingress-nginx-hull-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --secret-name=release-name-ingress-nginx-hull-admission
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: create
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
  ttlSecondsAfterFinished: 0
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: batch/v1
kind: Job
metadata:
  annotations:
    helm.sh/hook: post-install,post-upgrade
    helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
  labels:
    app.kubernetes.io/component: admissionpatch
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-admissionpatch
spec:
  template:
    metadata:
      annotations:
        helm.sh/hook: post-install,post-upgrade
        helm.sh/hook-delete-policy: before-hook-creation,hook-succeeded
      labels:
        app.kubernetes.io/component: admissionpatch
        app.kubernetes.io/instance: RELEASE-NAME
        app.kubernetes.io/managed-by: Helm
        app.kubernetes.io/name: ingress-nginx-hull
        app.kubernetes.io/part-of: undefined
        app.kubernetes.io/version: 1.0.4
        helm.sh/chart: ingress-nginx-hull-4.0.6
    spec:
      containers:
      - args:
        - patch
        - --webhook-name=release-name-ingress-nginx-hull-admission,release-name-ingress-nginx-hull-admission.$(POD_NAMESPACE).svc
        - --namespace=$(POD_NAMESPACE)
        - --patch-mutating=false
        - --secret-name=release-name-ingress-nginx-hull-admission
        - --patch-failure-policy=Fail
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        envFrom: []
        image: k8s.gcr.io/ingress-nginx/kube-webhook-certgen:v1.1.1
        imagePullPolicy: IfNotPresent
        name: patch
        ports: []
        volumeMounts: []
      imagePullSecrets: []
      initContainers: []
      nodeSelector:
        kubernetes.io/os: linux
      restartPolicy: OnFailure
      securityContext:
        runAsNonRoot: true
        runAsUser: 2000
      serviceAccountName: release-name-ingress-nginx-hull-default
      volumes: []
  ttlSecondsAfterFinished: 0
```

Great, last thing to do in this tutorial part is to convert the `templates/default-backend-deployment.yaml`:

```bash
$ cat ../ingress-nginx/templates/default-backend-deployment.yaml
```

```yaml
{{- if .Values.defaultBackend.enabled -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: default-backend
  name: {{ include "ingress-nginx.defaultBackend.fullname" . }}
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    matchLabels:
      {{- include "ingress-nginx.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: default-backend
{{- if not .Values.defaultBackend.autoscaling.enabled }}
  replicas: {{ .Values.defaultBackend.replicaCount }}
{{- end }}
  revisionHistoryLimit: {{ .Values.revisionHistoryLimit }}
  template:
    metadata:
    {{- if .Values.defaultBackend.podAnnotations }}
      annotations: {{ toYaml .Values.defaultBackend.podAnnotations | nindent 8 }}
    {{- end }}
      labels:
        {{- include "ingress-nginx.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: default-backend
      {{- if .Values.defaultBackend.podLabels }}
        {{- toYaml .Values.defaultBackend.podLabels | nindent 8 }}
      {{- end }}
    spec:
    {{- if .Values.imagePullSecrets }}
      imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
    {{- end }}
    {{- if .Values.defaultBackend.priorityClassName }}
      priorityClassName: {{ .Values.defaultBackend.priorityClassName }}
    {{- end }}
    {{- if .Values.defaultBackend.podSecurityContext }}
      securityContext: {{ toYaml .Values.defaultBackend.podSecurityContext | nindent 8 }}
    {{- end }}
      containers:
        - name: {{ template "ingress-nginx.name" . }}-default-backend
          {{- with .Values.defaultBackend.image }}
          image: "{{- if .repository -}}{{ .repository }}{{ else }}{{ .registry }}/{{ .image }}{{- end -}}:{{ .tag }}{{- if (.digest) -}} @{{.digest}} {{- end -}}"
          {{- end }}
          imagePullPolicy: {{ .Values.defaultBackend.image.pullPolicy }}
        {{- if .Values.defaultBackend.extraArgs }}
          args:
          {{- range $key, $value := .Values.defaultBackend.extraArgs }}
            {{- /* Accept keys without values or with false as value */}}
            {{- if eq ($value | quote | len) 2 }}
            - --{{ $key }}
            {{- else }}
            - --{{ $key }}={{ $value }}
            {{- end }}
          {{- end }}
        {{- end }}
          securityContext:
            capabilities:
              drop:
              - ALL
            runAsUser: {{ .Values.defaultBackend.image.runAsUser }}
            runAsNonRoot: {{ .Values.defaultBackend.image.runAsNonRoot }}
            allowPrivilegeEscalation: {{ .Values.defaultBackend.image.allowPrivilegeEscalation }}
            readOnlyRootFilesystem: {{ .Values.defaultBackend.image.readOnlyRootFilesystem}}
        {{- if .Values.defaultBackend.extraEnvs }}
          env: {{ toYaml .Values.defaultBackend.extraEnvs | nindent 12 }}
        {{- end }}
          livenessProbe:
            httpGet:
              path: /healthz
              port: {{ .Values.defaultBackend.port }}
              scheme: HTTP
            initialDelaySeconds: {{ .Values.defaultBackend.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.defaultBackend.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.defaultBackend.livenessProbe.timeoutSeconds }}
            successThreshold: {{ .Values.defaultBackend.livenessProbe.successThreshold }}
            failureThreshold: {{ .Values.defaultBackend.livenessProbe.failureThreshold }}
          readinessProbe:
            httpGet:
              path: /healthz
              port: {{ .Values.defaultBackend.port }}
              scheme: HTTP
            initialDelaySeconds: {{ .Values.defaultBackend.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.defaultBackend.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.defaultBackend.readinessProbe.timeoutSeconds }}
            successThreshold: {{ .Values.defaultBackend.readinessProbe.successThreshold }}
            failureThreshold: {{ .Values.defaultBackend.readinessProbe.failureThreshold }}
          ports:
            - name: http
              containerPort: {{ .Values.defaultBackend.port }}
              protocol: TCP
        {{- if .Values.defaultBackend.extraVolumeMounts }}
          volumeMounts: {{- toYaml .Values.defaultBackend.extraVolumeMounts | nindent 12 }}
        {{- end }}
        {{- if .Values.defaultBackend.resources }}
          resources: {{ toYaml .Values.defaultBackend.resources | nindent 12 }}
        {{- end }}
    {{- if .Values.defaultBackend.nodeSelector }}
      nodeSelector: {{ toYaml .Values.defaultBackend.nodeSelector | nindent 8 }}
    {{- end }}
      serviceAccountName: {{ template "ingress-nginx.defaultBackend.serviceAccountName" . }}
    {{- if .Values.defaultBackend.tolerations }}
      tolerations: {{ toYaml .Values.defaultBackend.tolerations | nindent 8 }}
    {{- end }}
    {{- if .Values.defaultBackend.affinity }}
      affinity: {{ toYaml .Values.defaultBackend.affinity | nindent 8 }}
    {{- end }}
      terminationGracePeriodSeconds: 60
    {{- if .Values.defaultBackend.extraVolumes }}
      volumes: {{ toYaml .Values.defaultBackend.extraVolumes | nindent 8 }}
    {{- end }}
{{- end }}
```

One way to transport this to your new chart is the following more compact input specification:

```bash
$ echo '      defaultbackend:
        enabled: _HT?(index . "$").Values.hull.config.specific.defaultBackend.enabled
        revisionHistoryLimit: 10
        pod:
          containers:
            defaultbackend:
              image:
                registry: k8s.gcr.io
                repository: defaultbackend-amd64
                tag: "1.5"
              imagePullPolicy: IfNotPresent
              securityContext:
                capabilities:
                  drop:
                  - ALL
                runAsUser: 65534
                runAsNonRoot: true
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
              livenessProbe:
                httpGet:
                  path: /healthz
                  port: 8080
                  scheme: HTTP
                failureThreshold: 3
                initialDelaySeconds: 30
                periodSeconds: 10
                successThreshold: 1
                timeoutSeconds: 5
              readinessProbe:
                httpGet:
                  path: /healthz
                  port: 8080
                  scheme: HTTP
                failureThreshold: 6
                initialDelaySeconds: 0
                periodSeconds: 5
                successThreshold: 1
                timeoutSeconds: 5
              ports: 
                http:
                  containerPort: 8080
                  protocol: TCP
          terminationGracePeriodSeconds: 60' >> values.yaml
```
