
## Preparation

In this Part of the tutorial workload objects are created. A workload object has a pod at its core in which containers run code. Most important workload types are Deployments, DaemonSets, StatefulSets, Jobs and CronJobs. 

First off we again copy over our working chart from the last tutorial part to a new folder in the same manner as we did before. Go to your working directory and execute:

```sh
cd ~/kubernetes-dashboard-hull && cp -R 04_rbac/ 05_workloads && cd 05_workloads/kubernetes-dashboard-hull
```

As before, to delete the HULL `hull.objects` from `values.full.yaml` and copy the current `hull.config` to the `values.yaml` type the following:

```sh
sed '/  objects:/Q' values.full.yaml > values.yaml
```

and verify the result:

```sh
cat values.yaml
```

looks like:

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
```

If yes, then you are good to start the next tutorial part.

## Examining existing workloads

Now having the preparation out of the way first find out which of those mentioned workload objects are present in the original chart:

```sh
find ../kubernetes-dashboard/templates -type f -iregex '.*\(deployment\|daemonset\|statefulset\|job\).*' | sort
```

which returns:

```yml
../kubernetes-dashboard/templates/deployment.yaml
```

Only the `dashboard` deployment but it will suffice to explain the process.

### Specifying workloads with HULL

Now check the `templates/deployment.yaml` Deployment:

```sh
cat ../kubernetes-dashboard/templates/deployment.yaml
```

which is defined as follows:

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

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
    {{- with .Values.annotations }}
    {{ toYaml . | nindent 4 }}
    {{- end }}
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    app.kubernetes.io/component: kubernetes-dashboard
    {{- with .Values.labels }}
    {{ toYaml . | nindent 4 }}
    {{- end }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
  selector:
    matchLabels:
{{ include "kubernetes-dashboard.matchLabels" . | nindent 6 }}
      app.kubernetes.io/component: kubernetes-dashboard
  template:
    metadata:
      annotations:
        {{- if .Values.podAnnotations }}
        {{- include "common.tplvalues.render" (dict "value" .Values.podAnnotations "context" $) | nindent 8 }}
        {{- end }}
      labels:
        {{- include "kubernetes-dashboard.labels" . | nindent 8 }}
        app.kubernetes.io/component: kubernetes-dashboard
        {{- if .Values.podLabels }}
        {{- include "common.tplvalues.render" (dict "value" .Values.podLabels "context" $) | nindent 8 }}
        {{- end }}
    spec:
{{- with .Values.securityContext }}
      securityContext:
{{ toYaml . | nindent 8 }}
{{- end }}
      serviceAccountName: {{ template "kubernetes-dashboard.serviceAccountName" . }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        args:
          - --namespace={{ .Release.Namespace }}
{{- if not .Values.protocolHttp }}
          - --auto-generate-certificates
{{- end }}
{{- if .Values.metricsScraper.enabled }}
          - --sidecar-host=http://127.0.0.1:8000
{{- else }}
          - --metrics-provider=none
{{- end }}
{{- with .Values.extraArgs }}
{{ toYaml . | nindent 10 }}
{{- end }}
{{- with .Values.extraEnv }}
        env:
{{ toYaml . | nindent 10 }}
{{- end }}
        ports:
{{- if .Values.protocolHttp }}
        - name: http
          containerPort: 9090
          protocol: TCP
{{- else }}
        - name: https
          containerPort: 8443
          protocol: TCP
{{- end }}
        volumeMounts:
        - name: kubernetes-dashboard-certs
          mountPath: /certs
          # Create on-disk volume to store exec logs
        - mountPath: /tmp
          name: tmp-volume
{{- with .Values.extraVolumeMounts }}
{{ toYaml . | nindent 8 }}
{{- end }}
        livenessProbe:
          httpGet:
{{- if .Values.protocolHttp }}
            scheme: HTTP
            path: /
            port: 9090
{{- else }}
            scheme: HTTPS
            path: /
            port: 8443
{{- end }}
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
          timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
{{- with .Values.resources }}
        resources:
{{ toYaml . | nindent 10 }}
{{- end }}
{{- with .Values.containerSecurityContext }}
        securityContext:
{{ toYaml . | nindent 10 }}
{{- end }}
{{- if .Values.metricsScraper.enabled }}
      - name: dashboard-metrics-scraper
        image: "{{ .Values.metricsScraper.image.repository }}:{{ .Values.metricsScraper.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
{{- with .Values.metricsScraper.args }}
        args:
{{ toYaml . | nindent 10 }}
{{- end }}
        ports:
          - containerPort: 8000
            protocol: TCP
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 8000
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
          timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume

{{- if .Values.metricsScraper.containerSecurityContext }}
        securityContext:
{{ toYaml .Values.metricsScraper.containerSecurityContext | nindent 10 }}
{{- else if .Values.containerSecurityContext}}
        securityContext:
{{ toYaml .Values.containerSecurityContext | nindent 10 }}
{{- end }}
{{- with .Values.metricsScraper.resources }}
        resources:
{{ toYaml . | nindent 10 }}
{{- end }}
{{- end }}
{{- with .Values.image.pullSecrets }}
      imagePullSecrets:
{{- range . }}
        - name: {{ . }}
{{- end }}
{{- end }}
{{- with .Values.nodeSelector }}
      nodeSelector:
{{ toYaml . | nindent 8 }}
{{- end }}
{{- with .Values.priorityClassName }}
      priorityClassName: "{{ . }}"
{{- end }}
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: {{ template "kubernetes-dashboard.fullname" . }}-certs
      - name: tmp-volume
        emptyDir: {}
{{- with .Values.extraVolumes }}
{{ toYaml . | nindent 6 }}
{{- end }}
{{- with .Values.tolerations }}
      tolerations:
{{ toYaml . | nindent 8 }}
{{- end }}
{{- with .Values.affinity }}
      affinity:
{{ toYaml . | nindent 8 }}
{{- end }}
```

Many Helm charts are full of 1:1 mappings between `values.yaml` properties and the target object's properties of the same type in the `/templates` files. Cases in point for the above Deployment are:

```yml
{{- with .Values.securityContext }}
      securityContext:
{{ toYaml . | nindent 8 }}
```
```yml
{{- with .Values.containerSecurityContext }}
        securityContext:
{{ toYaml . | nindent 10 }}
{{- end }}
```
```yml
{{- with .Values.metricsScraper.resources }}
        resources:
{{ toYaml . | nindent 10 }}
{{- end }}
```
```yml
{{- with .Values.image.pullSecrets }}
      imagePullSecrets:
{{- range . }}
        - name: {{ . }}
{{- end }}
{{- end }}
```
```yml
{{- with .Values.nodeSelector }}
      nodeSelector:
{{ toYaml . | nindent 8 }}
{{- end }}
```

There are a couple of more but the point of 1:1 mappings should be clear. 
Now here one of or maybe the major strength of HULL comes into play: setting any object property can be done directly without the need to write any repetitive boilerplate code in the sense of: 

> _if this value is provided in `values.yaml` do insert it here 'as is' in the template_

Using HULL all of the shown code blocks can be ignored for our conversion because we can configure the properties without any additional required action at configuration/deployment time if needed. No additional PR needed to make them configurable, they are always accessible if changing them is required.

### Automated `selector` population

Selectors for pods are automatically created by HULL and take the unique metadata combination of the labels:

- `app.kubernetes.io/component: default`
- `app.kubernetes.io/instance: release-name`
- `app.kubernetes.io/name: kubernetes-dashboard`

into account. With these fields being autopopulated at the workload and pod label level nothing further needs to be specified.

### Specifying the Pod `template`

These are the new and interesting aspects when investigating the Deployments pod specification we need to deal with:

#### The main container `name`

A quick note on the actual main container name which depends on the chart name in the original `kubernetes-dashboard` chart:

```yml
- name: {{ .Chart.Name }}
```

Here the container name will always resorte to `kubernetes-dashboard` but it may also be bound to some other value in the `values.yaml`. Since with HULL the name is derived by the container's key, changing the key in a system specific configuration would require to redefine all the containers content which is not desired. Hence you have to stick with the static defined container name `kubernetes-dashboard` in the HULL version of the chart but normally container names are of minimal importance to the Deployment.

#### The `imagePullSecrets` and `image.registry` feature

A word on `imagePullSecrets` though: it is very easy to configure and use them with HULL throughout all workloads which should be briefly highlighted. 

Firstly, HULL splits the image definition of containers into three parts which are concatenated to form the `image` value in the Pod spec:

- `registry`
- `repository`
- `tag`

where the leading registry part is optional.

Secondly there is a dedicated object type named `registry` where you can specify Docker Registry secrets. With dedicated `registry` fields in the `image` definition and the dedicated `registry` objeccts, HULL by default adds all `registry` objects that are created in HULL as `imagePullSecrets` to the pod specs so images are always pullable even if they are stored on a configured secured registry. If this behavior is undesired, you can disable this feature simply by setting `createImagePullSecretsFromRegistries` to false in the HULL configuration like this:

```yml
hull:
  config:
    general:
      createImagePullSecretsFromRegistries: false
```

You can however combine this feature with the possibility to set all pods `registry` fields to a common value. This is allowed for either using a fixed registry `server` name (for which the Docker registry secret is deployed outside of the Helm chart) or for using the `server` defined in the first found registry secret that is configured in the same chart. 

To provide an example of a real life use case imagine you host all Docker images referenced in your Helm chart in one place - meaning in one Docker registry. Now you may want to switch Docker registry providers - and hence the endpoint - or you need to copy over all Docker images to an airgapped system and host them in a local secured registry such as [Harbor](https://github.com/goharbor/harbor). In both cases you may want to centrally change the registry servers address without having to configure each pod specs `imagePullSecrets` and each containers `image` individually.

To use a static server name you can populate the `hull.config.general.defaultImageRegistryServer` field, this requires you to add the `imagePullSecrets` yourself to the pod specs. If you define the registry within the same chart you may just enable `hull.config.general.defaultImageRegistryToFirstRegistrySecretServer` and all is done automatically in terms of `imagePullSecrets` and `registry` population.

Refer to the [documentation](https://github.com/vidispine/hull/blob/main/hull/README.md#the-config-section) and the following `config` switches:

```yml
hull:
  config:
    general:
      defaultImageRegistryServer:
      defaultImageRegistryToFirstRegistrySecretServer:
```

for more details on how to work with this feature.

#### Dynamically creating `args`

The `args` section requires using a HULL transformation to create an array of arguments that match the dynamic nature of the argument construction.

#### Seperating out configuration to `hull.config.specific`

Multiple pod specific properties are referencing properties under the `.Values.metricsScraper` section in the original charts `values.yaml`:

```yml
{{- if .Values.metricsScraper.enabled }}
          - --sidecar-host=http://127.0.0.1:8000
{{- else }}
          - --metrics-provider=none
{{- end }}
```

and:

```yml
{{- if .Values.metricsScraper.enabled }}
      - name: dashboard-metrics-scraper
        image: "{{ .Values.metricsScraper.image.repository }}:{{ .Values.metricsScraper.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
{{- with .Values.metricsScraper.args }}
        args:
{{ toYaml . | nindent 10 }}
{{- end }}
        ports:
          - containerPort: 8000
            protocol: TCP
        livenessProbe:
          httpGet:
            scheme: HTTP
            path: /
            port: 8000
          initialDelaySeconds: {{ .Values.livenessProbe.initialDelaySeconds }}
          timeoutSeconds: {{ .Values.livenessProbe.timeoutSeconds }}
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume

{{- if .Values.metricsScraper.containerSecurityContext }}
        securityContext:
{{ toYaml .Values.metricsScraper.containerSecurityContext | nindent 10 }}
{{- else if .Values.containerSecurityContext}}
        securityContext:
{{ toYaml .Values.containerSecurityContext | nindent 10 }}
{{- end }}
{{- with .Values.metricsScraper.resources }}
        resources:
{{ toYaml . | nindent 10 }}
{{- end }}
{{- end }}
```

Basically it boils down to an optional sidecar container `dashboard-metrics-scraper` which may be enabled at will. The relevant section in `values.yaml`:

```sh
cat ../kubernetes-dashboard/values.yaml | grep "metricsScraper" -B 4 -A 17
```
looks as follows:

```yml

## Metrics Scraper
## Container to scrape, store, and retrieve a window of time from the Metrics Server.
## refs: https://github.com/kubernetes-sigs/dashboard-metrics-scraper
metricsScraper:
  ## Wether to enable dashboard-metrics-scraper
  enabled: false
  image:
    repository: kubernetesui/metrics-scraper
    tag: v1.0.7
  resources: {}
  ## SecurityContext especially for the kubernetes dashboard metrics scraper container
  ## If not set, the global containterSecurityContext values will define these values
  # containerSecurityContext:
  #   allowPrivilegeEscalation: false
  #   readOnlyRootFilesystem: true
  #   runAsUser: 1001
  #   runAsGroup: 2001
#  args:
#    - --log-level=info
#    - --logtostderr=true

```

To model this in your HULL chart you can just predefine the side car container and set `enabled: false` on it so it will behave the same as in the original chart. Regarding the reference in the `args` it can be hooked up with the `enabled` property. You may of course reference and specify the `metricsScraper.enabled` property in the `hull.config.specific` section to highlight the feature itself but it's not done in this tutorial. Most important for a proper Helm chart is to  document the `values.yaml` appropriately as to how the `metricsScraper` can be enabled, for the sake of brevity documentation and comments on the `values.yaml` are generally omitted.

Back to the pod specification, the `.Values.protocolHttp` property is also accessed multiple times too:

```yml
{{- if not .Values.protocolHttp }}
          - --auto-generate-certificates
{{- end }}
```

and:

```yml
{{- if .Values.protocolHttp }}
        - name: http
          containerPort: 9090
          protocol: TCP
{{- else }}
        - name: https
          containerPort: 8443
          protocol: TCP
{{- end }}
```

This property is suitable to be added in the `hull.config.specific` section to allow centrally switching to HTTP mode. Properties such as these can be considered custom more abstract properties that do not directly translate to one object property but control multiple aspects of maybe multiple objects even. Other properties are more concrete properties which are directly associated with an object property (e.g. `imagePullPolicy`, `affinity`, `tolerations`, ...)

The custom abstract properties are best modeled somewhere under `hull.config.specific` but for the concrete ones are best defined at deploy time by the configurator of the chart, adding redirection to `hull.config.specific` can clutter the configurational transparency.

But there are exceptions, e.g. consider a Helm chart allows to deploy it's application as either a Deployment or DaemonSet and shares a large amount of both objects configuration via a shared pod template definition:

  - if you aim for transparency it is best to define the properties directly on the DaemonSet and Deployment objects so the same property can have different values configured for both of them.

    > _Advantage here is that it is more straightforward to understand where each object's property value is defined if at all and how it can be changed._

    > _Disadvantage is that this weakens the likely intention to have the DaemonSet and Deployment as alike as possible since it allows to involuntarily have unwanted differences between them._

  - if you aim for abstraction you should define all the properties that are being referenced from both Deplyoment and DaemonSet under the `hull.config.specific` section and reference the central values.

    > _Disadvantage is that it obfuscates where a properties value comes from more and requires much more effort to set up the chart._

    > _Advantages are that the intent of having DaemonSet and Deployment as much alike as possible is better fulfillable and it would still be possible to overwrite any individual property on object level if wanted._

#### Putting it all together

Ok so first insert the new `hull.config.specific` property so it can be referenced to:

```sh
sed '/^\s\s\s\sspecific/r'<(
      echo "      protocolHttp: false"
    ) -i -- values.yaml
```

Good, now when writing the Deployment spec default property values can be taken from the relevant fields in the original `values.yaml` where they are provided. For example, the `values.yaml` holds defaults for `securityContext`'s:

```sh
cat ../kubernetes-dashboard/values.yaml | grep "securityContext" -B 3 -A 12
```

of the pod and the main container and a commented out suggestion for the `metricsScraper` container:

```yml

## SecurityContext to be added to kubernetes dashboard pods
## To disable set the following configuration to null:
# securityContext: null
securityContext:
  seccompProfile:
    type: RuntimeDefault

## SecurityContext defaults for the kubernetes dashboard container and metrics scraper container
## To disable set the following configuration to null:
# containerSecurityContext: null
containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsUser: 1001
  runAsGroup: 2001

--
  ## Maximum unavailable instances; ignored if there is no PodDisruptionBudget
  maxUnavailable:

## PodSecurityContext for pod level securityContext
# securityContext:
#   runAsUser: 1001
#   runAsGroup: 2001

networkPolicy:
  # Whether to create a network policy that allows/restricts access to the service
  enabled: false

## podSecurityPolicy for fine-grained authorization of pod creation and updates
podSecurityPolicy:
  # Specifies whether a pod security policy should be created
  enabled: false

```

Many more fields that have simple defaults in the `kubernetes-dashboard`'s `values.yaml` can be copied over to our new HULL based Deployment specification, amongst them `image`, `resources` and `imagePullSecrets`.

Other YAML structures in the Deployment are mixtures of static content and templated parts, take a look at the original `args`:

```yml
        args:
          - --namespace={{ .Release.Namespace }}
{{- if not .Values.protocolHttp }}
          - --auto-generate-certificates
{{- end }}
{{- if .Values.metricsScraper.enabled }}
          - --sidecar-host=http://127.0.0.1:8000
{{- else }}
          - --metrics-provider=none
{{- end }}
{{- with .Values.extraArgs }}
{{ toYaml . | nindent 10 }}
{{- end }}
```

or are build up completely based on templating decisions like the `ports`:

```yml
        ports:
{{- if .Values.protocolHttp }}
        - name: http
          containerPort: 9090
          protocol: TCP
{{- else }}
        - name: https
          containerPort: 8443
          protocol: TCP
{{- end }}
```

To replicate this behavior you can use the `_HT!` HULL transformation to completely model the arrays or dictionaries as desired or - in case of the simpler either/or decision for the `ports` - you can utilize hooking up the individual port definitions `enabled` property to a condition as shown next. HULL offers flexibility to model the configuration as you think it fits best.

So this is a proposition for modeling the suggested Deployment spec for the `kubernetes-dashboard-hull` chart:

```sh
echo '  objects:
    deployment:
      dashboard:
        enabled: true
        replicas: 1
        strategy:
          rollingUpdate:
            maxSurge: 0
            maxUnavailable: 1
          type: RollingUpdate
        pod:
          securityContext:
            seccompProfile:
              type: RuntimeDefault
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: v2.5.0
              imagePullPolicy: IfNotPresent
              args: _HT![
                  {{- $p := (index . "$") }}
                  "--namespace={{ $p.Release.Namespace }}",
                  {{- if not $p.Values.hull.config.specific.protocolHttp }}
                  "--auto-generate-certificates",
                  {{- end }}
                  {{- if $p.Values.hull.objects.deployment.dashboard.pod.containers.metricsscraper.enabled }}
                  "--sidecar-host=http://127.0.0.1:8000",
                  {{- else }}
                  "--metrics-provider=none",
                  {{- end }}
                  ]
              ports:
                http:
                  enabled: _HT?(index . "$").Values.hull.config.specific.protocolHttp
                  containerPort: 9090
                  protocol: TCP
                https:
                  enabled: _HT?(not (index . "$").Values.hull.config.specific.protocolHttp)
                  containerPort: 8443
                  protocol: TCP
              volumeMounts:
                dashboard:
                  name: kubernetes-dashboard-certs
                  mountPath: /certs
                tmp:
                  name: tmp-volume
                  mountPath: /tmp
              livenessProbe: |-
                _HT!{
                  httpGet: {
                    path: /,
                    scheme: {{ (index . "$").Values.hull.config.specific.protocolHttp | ternary "HTTP" "HTTPS" }},
                    port: {{ (index . "$").Values.hull.config.specific.protocolHttp | ternary 9090 8443 }}
                    }
                  }
                initialDelaySeconds: 30
                timeoutSeconds: 30
              resources:
                requests:
                  cpu: 100m
                  memory: 200Mi
                limits:
                  cpu: 2
                  memory: 200Mi
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                runAsUser: 1001
                runAsGroup: 2001
            metricsscraper:
              enabled: false
              image:
                repository: kubernetesui/metrics-scraper
                tag: v1.0.7
              imagePullPolicy: IfNotPresent
              ports:
                tcp:
                  containerPort: 8000
                  protocol: TCP
              livenessProbe:
                httpGet:
                  scheme: HTTP
                  path: /
                  port: 8000
                initialDelaySeconds: 30
                timeoutSeconds: 30
              volumeMounts:
                tmp:
                  name: tmp-volume
                  mountPath: /tmp
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                runAsUser: 1001
                runAsGroup: 2001
          volumes:
            kubernetes-dashboard-certs:
              secret:
                secretName: certs
            tmp-volume:
              emptyDir: {}' >> values.yaml
```

Does it render as expected?

```sh
helm template .
```

yields:

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
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
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
      - args:
        - --namespace=default
        - --auto-generate-certificates
        - --metrics-provider=none
        env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /
            port: 8443
            scheme: HTTPS
        name: dashboard
        ports:
        - containerPort: 8443
          name: https
          protocol: TCP
        resources:
          limits:
            cpu: 2
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsGroup: 2001
          runAsUser: 1001
        volumeMounts:
        - mountPath: /certs
          name: kubernetes-dashboard-certs
        - mountPath: /tmp
          name: tmp-volume
      imagePullSecrets: []
      initContainers: []
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      serviceAccountName: release-name-kubernetes-dashboard-default
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: release-name-kubernetes-dashboard-certs
      - emptyDir: {}
        name: tmp-volume
```

Everything should be as expected. How about switching to HTTP and enabling the `metricsscraper` for a test?

```sh
echo 'hull:
  config:
    specific:
      protocolHttp: true
  objects:
    deployment:
      dashboard:
        pod:
          containers:
            metricsscraper:
              enabled: true' > ../configs/enable-http-scraper.yaml
```

Check the result:

```sh
helm template -f ../configs/enable-http-scraper.yaml .
```

and you'll see that the `args`, `ports` and `livenessProbe` output has changed to reflect the change to the `protocolHttp` property and the `metricsscraper` container is activated:

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
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
    type: RollingUpdate
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
      - args:
        - --namespace=default
        - --sidecar-host=http://127.0.0.1:8000
        env: []
        envFrom: []
        image: kubernetesui/dashboard:v2.5.0
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /
            port: 9090
            scheme: HTTP
        name: dashboard
        ports:
        - containerPort: 9090
          name: http
          protocol: TCP
        resources:
          limits:
            cpu: 2
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsGroup: 2001
          runAsUser: 1001
        volumeMounts:
        - mountPath: /certs
          name: kubernetes-dashboard-certs
        - mountPath: /tmp
          name: tmp-volume
      - env: []
        envFrom: []
        image: kubernetesui/metrics-scraper:v1.0.7
        imagePullPolicy: IfNotPresent
        livenessProbe:
          httpGet:
            path: /
            port: 8000
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 30
        name: metricsscraper
        ports:
        - containerPort: 8000
          name: tcp
          protocol: TCP
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsGroup: 2001
          runAsUser: 1001
        volumeMounts:
        - mountPath: /tmp
          name: tmp-volume
      imagePullSecrets: []
      initContainers: []
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      serviceAccountName: release-name-kubernetes-dashboard-default
      volumes:
      - name: kubernetes-dashboard-certs
        secret:
          secretName: release-name-kubernetes-dashboard-certs
      - emptyDir: {}
        name: tmp-volume
```

## Wrap Up

Thanks for your interest, your `values.yaml` is supposed to look like this at this point:

```yml
metrics-server:
  enabled: false
hull:
  config:
    specific:
      protocolHttp: false
      rbac:
        clusterReadOnlyRole: false
        clusterRoleMetrics: true
      settings: {}
      pinnedCRDs: {}
  objects:
    deployment:
      dashboard:
        enabled: true
        replicas: 1
        strategy:
          rollingUpdate:
            maxSurge: 0
            maxUnavailable: 1
          type: RollingUpdate
        pod:
          securityContext:
            seccompProfile:
              type: RuntimeDefault
          containers:
            dashboard:
              image:
                repository: kubernetesui/dashboard
                tag: v2.5.0
              imagePullPolicy: IfNotPresent
              args: _HT![
                  {{- $p := (index . "$") }}
                  "--namespace={{ $p.Release.Namespace }}",
                  {{- if not $p.Values.hull.config.specific.protocolHttp }}
                  "--auto-generate-certificates",
                  {{- end }}
                  {{- if $p.Values.hull.objects.deployment.dashboard.pod.containers.metricsscraper.enabled }}
                  "--sidecar-host=http://127.0.0.1:8000",
                  {{- else }}
                  "--metrics-provider=none",
                  {{- end }}
                  ]
              ports:
                http:
                  enabled: _HT?(index . "$").Values.hull.config.specific.protocolHttp
                  containerPort: 9090
                  protocol: TCP
                https:
                  enabled: _HT?(not (index . "$").Values.hull.config.specific.protocolHttp)
                  containerPort: 8443
                  protocol: TCP
              volumeMounts:
                dashboard:
                  name: kubernetes-dashboard-certs
                  mountPath: /certs
                tmp:
                  name: tmp-volume
                  mountPath: /tmp
              livenessProbe: |-
                _HT!{
                  httpGet: {
                    path: /,
                    scheme: {{ (index . "$").Values.hull.config.specific.protocolHttp | ternary "HTTP" "HTTPS" }},
                    port: {{ (index . "$").Values.hull.config.specific.protocolHttp | ternary 9090 8443 }}
                    }
                  }
                initialDelaySeconds: 30
                timeoutSeconds: 30
              resources:
                requests:
                  cpu: 100m
                  memory: 200Mi
                limits:
                  cpu: 2
                  memory: 200Mi
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                runAsUser: 1001
                runAsGroup: 2001
            metricsscraper:
              enabled: false
              image:
                repository: kubernetesui/metrics-scraper
                tag: v1.0.7
              imagePullPolicy: IfNotPresent
              ports:
                tcp:
                  containerPort: 8000
                  protocol: TCP
              livenessProbe:
                httpGet:
                  scheme: HTTP
                  path: /
                  port: 8000
                initialDelaySeconds: 30
                timeoutSeconds: 30
              volumeMounts:
                tmp:
                  name: tmp-volume
                  mountPath: /tmp
              securityContext:
                allowPrivilegeEscalation: false
                readOnlyRootFilesystem: true
                runAsUser: 1001
                runAsGroup: 2001
          volumes:
            kubernetes-dashboard-certs:
              secret:
                secretName: certs
            tmp-volume:
              emptyDir: {}
```

Again backup your finished `values.yaml` to the `values.tutorial-part.yaml` in case you want to modify, play around or start from scratch again with the `values.yaml`:

```sh
cp values.yaml values.tutorial-part.yaml
```

and add in the objects things created in the previous tutorial:

```sh
sed '1,/objects:/d' values.full.yaml > _tmp && cp values.yaml values.full.yaml && cat _tmp >> values.full.yaml && rm _tmp
```