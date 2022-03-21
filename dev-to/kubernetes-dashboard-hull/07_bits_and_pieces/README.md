
## Preparation

In this tutorial section you are going to deal with the remaining template files and objects from the `kubernetes-dashboard` chart which we haven't covered yet. This will conclude the educational conversion of objects from a 'regular' Helm chart to a HULL based chart.

But before diving into this prepare your working folder a last time:

Copy over the last sections result:

```sh
cd ~/kubernetes-dashboard-hull && cp -R 06_service_api/ 07_bits_and_pieces && cd 07_bits_and_pieces/kubernetes-dashboard-hull
```

Prepare the `values.yaml`:

```sh
sed '/  objects:/Q' values.full.yaml > values.yaml
```

and once more verify the result:

```sh
cat values.yaml
```

currently looks like:

```yml
metrics-server:
  enabled: false
hull:
  config:
    specific:
      externalPort: 443
      protocolHttp: false
      rbac:
        clusterReadOnlyRole: false
        clusterRoleMetrics: true
      settings: {}
      pinnedCRDs: {}
```

For a reminder of the source template check the templates folder again:

```sh
ls ../kubernetes-dashboard/templates
```

showing the following files:

```yml
NOTES.txt       clusterrole-metrics.yaml         clusterrolebinding-readonly.yaml  ingress.yaml        psp.yaml          secret.yaml          servicemonitor.yaml
_helpers.tpl    clusterrole-readonly.yaml        configmap.yaml                    networkpolicy.yaml  role.yaml         service.yaml
_tplvalues.tpl  clusterrolebinding-metrics.yaml  deployment.yaml                   pdb.yaml            rolebinding.yaml  serviceaccount.yaml
```

Most of them are covered by now, the remaining ones and subject of this tutorial section are:

```yml
networkpolicy.yaml
pdb.yaml
psp.yaml
servicemonitor.yaml
```

## Specifying NetworkPolicies

The `networkpolicy.yaml`:

```sh
cat ../kubernetes-dashboard/templates/networkpolicy.yaml
```

is pretty lightweight:

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

{{ if .Values.networkPolicy.enabled -}}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}
  labels:
    app: {{ template "kubernetes-dashboard.name" . }}
    chart: {{ template "kubernetes-dashboard.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
spec:
  podSelector:
    matchLabels:
{{ include "kubernetes-dashboard.matchLabels" . | nindent 6 }}
  ingress:
  - ports:
{{- if .Values.protocolHttp }}
    - port: http
      protocol: TCP
{{- else }}
    - port: https
      protocol: TCP
{{- end -}}
{{- end -}}
```

The addition of new fixed labels here:

```yml
  labels:
    app: {{ template "kubernetes-dashboard.name" . }}
    chart: {{ template "kubernetes-dashboard.chart" . }}
    release: {{ .Release.Name }}
    heritage: {{ .Release.Service }}
```

appears to be out of line with the metadata of the remaining objects in this chart and may likely not have a functionality even. Anyway for completeness sake add them anyway to the new objects metadata labels by transporting the underlying logic.

Regarding the `podSelector` you can utilize HULL's `hull.tranformation.selector` function to create the corresponding labels to match the `dashboard` pods. 

That's all, write it to the `values.yaml`:

```sh
echo '  objects:
    networkpolicy:
      dashboard:
        labels:
          app: _HT!{{- default (index . "$").Chart.Name (index . "$").Values.nameOverride | trunc 63 | trimSuffix "-" -}}
          chart: _HT!{{- printf "%s-%s" (index . "$").Chart.Name (index . "$").Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
          release: _HT!{{ (index . "$").Release.Name }}
          heritage: _HT!{{ (index . "$").Release.Service }}
        podSelector:
          matchLabels: _HT&dashboard' >> values.yaml
```

apply the `helm template` command:

```sh
helm template .
```

and ponder the results:

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations: {}
  labels:
    app: kubernetes-dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    chart: kubernetes-dashboard-5.2.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
    heritage: Helm
    release: release-name
  name: release-name-kubernetes-dashboard-dashboard
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
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

## Specifying a PodDisruptionBudget

[PodDisruptionBudgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) have three fields:

- A label selector .spec.selector to specify the set of pods to which it applies. This field is required.
- .spec.minAvailable which is a description of the number of pods from that set that must still be available after the eviction, even in the absence of the evicted pod. minAvailable can be either an absolute number or a percentage.
- .spec.maxUnavailable (available in Kubernetes 1.7 and higher) which is a description of the number of pods from that set that can be unavailable after the eviction. It can be either an absolute number or a percentage.


Take a look at our PodDisruptionBudget definition:

```sh
cat ../kubernetes-dashboard/templates/pdb.yaml
```

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

{{ if .Values.podDisruptionBudget.enabled -}}
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
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
  name: {{ template "kubernetes-dashboard.fullname" . }}
spec:
  {{- if .Values.podDisruptionBudget.minAvailable }}
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  {{- end  }}
  {{- if .Values.podDisruptionBudget.maxUnavailable }}
  maxUnavailable: {{ .Values.podDisruptionBudget.maxUnavailable }}
  {{- end  }}
  selector:
    matchLabels:
{{ include "kubernetes-dashboard.matchLabels" . | nindent 6 }}
{{- end -}}
```

In the `values.yaml`:

```sh
cat ../kubernetes-dashboard/values.yaml | grep "^podDisruptionBudget:" -B 3 -A 6
```

default values are omitted for `minAvailable` and `maxUnavailable`:

```yml

## podDisruptionBudget
## ref: https://kubernetes.io/docs/tasks/run-application/configure-pdb/
podDisruptionBudget:
  enabled: false
  ## Minimum available instances; ignored if there is no PodDisruptionBudget
  minAvailable:
  ## Maximum unavailable instances; ignored if there is no PodDisruptionBudget
  maxUnavailable:
```

Again you can put the handy `_HT&` selector transformation to good use here and specify the object template:

```sh
echo '    poddisruptionbudget:
      dashboard:
        enabled: false
        selector:
          matchLabels: _HT&dashboard' >> values.yaml
```

and do a quick check:

```sh
echo 'hull:
  objects:
    poddisruptionbudget:
      dashboard:
        enabled: true' > ../configs/pdb.yaml
```

of the rendered object:

```sh
helm template -f ../configs/pdb.yaml .
```

which is pretty bare bones:

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations: {}
  labels:
    app: kubernetes-dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    chart: kubernetes-dashboard-5.2.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
    heritage: Helm
    release: release-name
  name: release-name-kubernetes-dashboard-dashboard
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
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

## Setting up PodSecurityPolicies

Inspect the last remaining file:

```sh
cat ../kubernetes-dashboard/templates/psp.yaml
```

wherein you find a PodSecurityPolicy and surprisingly another Role and RoleBinding:

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

{{ if .Values.podSecurityPolicy.enabled -}}
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}-psp
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
spec:
  privileged: false
  fsGroup:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  runAsGroup:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
    - 'configMap'
    - 'secret'
    - 'emptyDir'
  allowPrivilegeEscalation: false
  hostNetwork: false
  hostIPC: false
  hostPID: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}-psp
  labels:
{{ include "kubernetes-dashboard.labels" . | nindent 4 }}
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: {{ template "kubernetes-dashboard.fullname" . }}-psp
subjects:
- kind: ServiceAccount
  name: {{ template "kubernetes-dashboard.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}-psp
  labels:
{{ include "kubernetes-dashboard.labels" . | nindent 4 }}
rules:
- apiGroups:
  - extensions
  - policy/v1beta1
  resources:
  - podsecuritypolicies
  verbs:
  - use
  resourceNames:
  - {{ template "kubernetes-dashboard.fullname" . }}-psp
{{- end -}}
```

Apart from the surprise of the unexpected objects there is nothing new to be found here and we can straight up convert them by adding the PodSecurityPolicy and the RoleBinding:

```sh
echo '    podsecuritypolicy:
      dashboard:
        enabled: false
        annotations:
          seccomp.security.alpha.kubernetes.io/allowedProfileNames: "*"
        privileged: false
        fsGroup:
          rule: RunAsAny
        runAsUser:
          rule: RunAsAny
        runAsGroup:
          rule: RunAsAny
        seLinux:
          rule: RunAsAny
        supplementalGroups:
          rule: RunAsAny
        volumes:
        - configMap
        - secret
        - emptyDir
        allowPrivilegeEscalation: false
        hostNetwork: false
        hostIPC: false
        hostPID: false
    rolebinding:
      psp:
        enabled: _HT?(index . "$").Values.hull.objects.podsecuritypolicy.dashboard.enabled
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: _HT^psp
        subjects:
        - kind: ServiceAccount
          namespace: _HT!{{ (index . "$").Release.Namespace }}
          name: _HT^default
    role:
      psp:
        enabled: _HT?(index . "$").Values.hull.objects.podsecuritypolicy.dashboard.enabled
        rules:
          psp:
            apiGroups:
            - extensions
            - policy/v1beta1
            resources:
            - podsecuritypolicies
            verbs:
            - use
            resourceNames: _HT![ {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "settings") }} ]' >> values.yaml
```

Enable the PSP with this:

```sh
echo 'hull:
  objects:
    podsecuritypolicy:
      dashboard:
        enabled: true' > ../configs/psp.yaml
```

and it is added alongside the Role and RoleBinding:

```sh
helm template -f ../configs/psp.yaml .
```

```yml
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  annotations: {}
  labels:
    app: kubernetes-dashboard
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    chart: kubernetes-dashboard-5.2.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
    heritage: Helm
    release: release-name
  name: release-name-kubernetes-dashboard-dashboard
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: dashboard
      app.kubernetes.io/instance: release-name
      app.kubernetes.io/name: kubernetes-dashboard
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  annotations:
    seccomp.security.alpha.kubernetes.io/allowedProfileNames: '*'
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
  allowPrivilegeEscalation: false
  fsGroup:
    rule: RunAsAny
  hostIPC: false
  hostNetwork: false
  hostPID: false
  privileged: false
  runAsGroup:
    rule: RunAsAny
  runAsUser:
    rule: RunAsAny
  seLinux:
    rule: RunAsAny
  supplementalGroups:
    rule: RunAsAny
  volumes:
  - configMap
  - secret
  - emptyDir
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
kind: Role
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: psp
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-psp
rules:
- apiGroups:
  - extensions
  - policy/v1beta1
  resourceNames:
  - release-name-kubernetes-dashboard-settings
  resources:
  - podsecuritypolicies
  verbs:
  - use
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
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: psp
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-psp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: release-name-kubernetes-dashboard-psp
subjects:
- kind: ServiceAccount
  name: release-name-kubernetes-dashboard-default
  namespace: default
---
```

## ServiceMonitor

```sh
echo '    servicemonitor:
      dashboard:
        enabled: false
        endpoints: _HT![ 
          {{ if (index . "$").Values.hull.config.specific.protocolHttp }}
          { port: "http" }
          {{ else }}
          { port: "https" }
          {{ end }}
         ]
        selector:
          matchLabels: _HT&dashboard' >> values.yaml
```

Enable the PSP with this:

```sh
echo 'hull:
  objects:
    servicemonitor:
      dashboard:
        enabled: true' > ../configs/servicemonitor.yaml
```

and it is added alongside the Role and RoleBinding:

```sh
helm template -f ../configs/servicemonitor.yaml .
```

```yml
## Another Wrap Up

The object conversion is done, time to clean up once more.

1. Your `values.yaml` should resemble this now:

    ```yml
    metrics-server:
  enabled: false
hull:
  config:
    specific:
      externalPort: 443
      protocolHttp: false
      rbac:
        clusterReadOnlyRole: false
        clusterRoleMetrics: true
      settings: {}
      pinnedCRDs: {}
  objects:
    networkpolicy:
      dashboard:
        labels:
          app: _HT!{{- default (index . "$").Chart.Name (index . "$").Values.nameOverride | trunc 63 | trimSuffix "-" -}}
          chart: _HT!{{- printf "%s-%s" (index . "$").Chart.Name (index . "$").Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
          release: _HT!{{ (index . "$").Release.Name }}
          heritage: _HT!{{ (index . "$").Release.Service }}
        podSelector:
          matchLabels: _HT&dashboard
    poddisruptionbudget:
      dashboard:
        enabled: false
        selector:
          matchLabels: _HT&dashboard
    podsecuritypolicy:
      dashboard:
        enabled: false
        annotations:
          seccomp.security.alpha.kubernetes.io/allowedProfileNames: "*"
        privileged: false
        fsGroup:
          rule: RunAsAny
        runAsUser:
          rule: RunAsAny
        runAsGroup:
          rule: RunAsAny
        seLinux:
          rule: RunAsAny
        supplementalGroups:
          rule: RunAsAny
        volumes:
        - configMap
        - secret
        - emptyDir
        allowPrivilegeEscalation: false
        hostNetwork: false
        hostIPC: false
        hostPID: false
    rolebinding:
      psp:
        enabled: _HT?(index . "$").Values.hull.objects.podsecuritypolicy.dashboard.enabled
        roleRef:
          apiGroup: rbac.authorization.k8s.io
          kind: Role
          name: _HT^psp
        subjects:
        - kind: ServiceAccount
          namespace: _HT!{{ (index . "$").Release.Namespace }}
          name: _HT^default
    role:
      psp:
        enabled: _HT?(index . "$").Values.hull.objects.podsecuritypolicy.dashboard.enabled
        rules:
          psp:
            apiGroups:
            - extensions
            - policy/v1beta1
            resources:
            - podsecuritypolicies
            verbs:
            - use
            resourceNames: _HT![ {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "settings") }} ]
    ```

2. Backup what you did in this tutorial part:

    ```sh
    cp values.yaml values.tutorial-part.yaml
    ```

3. Since there exists a `role` block now in the current `values.full.yaml` and our local `values.yaml` it is required to merge them appropriately now. Otherwise the second `role` key will overwrite the first which is not helpful. To create the final `values.full.yaml` you hence need to do a little more work but you can just copy and paste the following 'one lines' and it should do:

    ```sh
    sed '1,/objects:/d' values.full.yaml > _tmp && sed '/^\s\s\s\srole:/r'<(
      echo "      psp:"
      echo "        enabled: _HT?(index . \"$\").Values.hull.objects.podsecuritypolicy.dashboard.enabled"
      echo "        rules:"
      echo "          psp:"
      echo "            apiGroups:"
      echo "            - extensions"
      echo "            - policy/v1beta1"
      echo "            resources:"
      echo "            - podsecuritypolicies"
      echo "            verbs:"
      echo "            - use"
      echo "            resourceNames: _HT![ {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "settings") }} ]"
    ) -i -- _tmp && cp values.yaml values.full.yaml && sed '/^\s\s\s\srole:/,$d' -i values.full.yaml && cat _tmp >> values.full.yaml && rm _tmp
    ```