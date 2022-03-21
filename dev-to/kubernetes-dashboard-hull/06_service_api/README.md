
## Preparation

Next up is a closer look at the objects from the [Kubernetes Service API](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/) which includes Services, Ingresses and Endpoints.

But as before copy over our working chart from the last tutorial part to a new folder:

```sh
cd ~/kubernetes-dashboard-hull && cp -R 05_workloads/ 06_service_api && cd 06_service_api/kubernetes-dashboard-hull
```

To remove the HULL `hull.objects` from `values.full.yaml` and prepare our testing `values.yaml` enter:

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
      protocolHttp: false
      rbac:
        clusterReadOnlyRole: false
        clusterRoleMetrics: true
      settings: {}
      pinnedCRDs: {}
```

## Check for existing Service API objects

Start the investigation as you did before:

```sh
find ../kubernetes-dashboard/templates -type f -iregex '.*\(Service\|\Ingress\|\Endpoint\).*' | sort
```

which returns:

```yml
../kubernetes-dashboard/templates/ingress.yaml
../kubernetes-dashboard/templates/service.yaml
../kubernetes-dashboard/templates/serviceaccount.yaml
../kubernetes-dashboard/templates/servicemonitor.yaml
```

Ignoring the ServiceAccount (already handled) and the ServiceMonitor (handled later) there exists a Service and an Ingress definition.

## Defining Services in HULL

The Service template contents of `/templates/service.yaml`:

```sh
cat ../kubernetes-dashboard/templates/service.yaml
```

do look like this:

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

apiVersion: v1
kind: Service
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}
  labels:
    {{ include "kubernetes-dashboard.labels" . | nindent 4 }}
    app.kubernetes.io/component: kubernetes-dashboard
    {{ .Values.service.clusterServiceLabel.key | nindent 4}}: {{ .Values.service.clusterServiceLabel.enabled | quote }}
    {{- if .Values.service.labels }}
    {{ toYaml .Values.service.labels | nindent 4 }}
    {{- end }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
    {{- if .Values.service.annotations }}
    {{ toYaml .Values.service.annotations | nindent 4 }}
    {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
  - port: {{ .Values.service.externalPort }}
{{- if .Values.protocolHttp }}
    targetPort: http
    name: http
{{- else }}
    targetPort: https
    name: https
{{- end }}
{{- if hasKey .Values.service "nodePort" }}
    nodePort: {{ .Values.service.nodePort }}
{{- end }}
{{- if .Values.service.loadBalancerSourceRanges }}
  loadBalancerSourceRanges:
{{ toYaml .Values.service.loadBalancerSourceRanges | nindent 4 }}
{{- end }}
  selector:
{{ include "kubernetes-dashboard.matchLabels" . | nindent 4 }}
    app.kubernetes.io/component: kubernetes-dashboard
{{- if .Values.service.loadBalancerIP }}
  loadBalancerIP: {{ .Values.service.loadBalancerIP }}
{{- end }}
```

Again there is a reference to the `.Values.protocolHttp` property which now exists in the `hull.config.specific` section. The `ports`' `name` and `targetPort` are set dependend on `protocolHttp` while the `externalPort` is centrally defined. Therefore it makes sense to centralize access to `externalPort` under `hull.config.specific`, even more so because the `externalPort` is also accessed when defining the Ingress (more on that later). To do so execute:

```sh
sed '/^\s\s\s\sspecific/r'<(
      echo "      externalPort: 443"
    ) -i -- values.yaml
```

Apart from the `port`s there is only the `selector` worth discussing, the rest of the Service template is straightforward 1:1 mappings. But also check on the service block defaults in the `kubernetes-dashboard`'s `values.yaml` with:

```sh
cat ../kubernetes-dashboard/values.yaml | grep "^service:" -B 1 -A 23
```
which prints:

```yml

service:
  type: ClusterIP
  # Dashboard service port
  externalPort: 443

  ## LoadBalancerSourcesRange is a list of allowed CIDR values, which are combined with ServicePort to
  ## set allowed inbound rules on the security group assigned to the master load balancer
  # loadBalancerSourceRanges: []

  ## A user-specified IP address for load balancer to use as External IP (if supported)
  # loadBalancerIP:

  ## Additional Kubernetes Dashboard Service annotations
  annotations: {}

  ## Here labels can be added to the Kubernetes Dashboard service
  labels: {}

  ## Enable or disable the kubernetes.io/cluster-service label. Should be disabled for GKE clusters >=1.15.
  ## Otherwise, the addon manager will presume ownership of the service and try to delete it.
  clusterServiceLabel:
    enabled: true
    key: "kubernetes.io/cluster-service"
```

The label `kubernetes.io/cluster-service` is added to the Service object and the `enabled` value serves as the value, this is represented in this part of the Service template:

```yml
    {{ .Values.service.clusterServiceLabel.key | nindent 4}}: {{ .Values.service.clusterServiceLabel.enabled | quote }}
```

While you could mimic the logic and move the `clusterServiceLabel` to the `hull.config.specific` section it does not offer any advantage over adding the label to the new service instance with the default value of `true`. A comment may be added of course to explain the deeper meaning  of this label.

### A closer look at `selector`s of Services

As previously highlighted, the `selector` `matchLabels` for workloads are automatically constructed from the following labels:

- `app.kubernetes.io/component: default`
- `app.kubernetes.io/instance: release-name`
- `app.kubernetes.io/name: kubernetes-dashboard`

The Service object `selector`'s in HULL follow the same construction principle, so this means that any Service whose key is identical to a workload object instance the `selector` will match and the Service will front the pods of the workload. For our example, if you define either a Deployment, DaemonSet or StatefulSet with key `dashboard`, any Service object with the same key `dashboard` will match and pose as the Service object to the pods on the network level. This is what you want to achieve here, a Service for the `dashboard` Deployment. On a side note, for Jobs the `selector` property is omitted automatically by HULL because in this case the `selector` handling of Jobs is internally managed by Kubernetes.

So if you leave out the `selector` specification you will automatically have the default `selector` created but if you want to model the `selector` property yourself you can overwrite it explicitly. A use case for this is e.g. the creation of an additional headless Service for StatefulSets or any other more finegrained controlling of Service to Pod matching. For this there also exists a HULL transformation, the `hull.util.transformation.selector` or short `_HT&`, which will be put to use later on. 

But for now here is the converted Service object for the `dashboard` employing the techniques that you already have learned:

```sh
echo '  objects:
    service:
      dashboard:
        labels:
          ## Enable or disable the kubernetes.io/cluster-service label. Should be disabled for GKE clusters >=1.15.
          ## Otherwise, the addon manager will presume ownership of the service and try to delete it.
          "kubernetes.io/cluster-service": "true"
        type: ClusterIP
        ports:
          http:
            enabled: _HT?(index . "$").Values.hull.config.specific.protocolHttp
            port: _HT*hull.config.specific.externalPort
            targetPort: http
          https:
            enabled: _HT?(not (index . "$").Values.hull.config.specific.protocolHttp)
            port: _HT*hull.config.specific.externalPort
            targetPort: https' >> values.yaml
```

## Defining Ingresses for inbound traffic

Next take a look at the original ingress definition with:

```sh
cat ../kubernetes-dashboard/templates/ingress.yaml
```

returning:

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

{{ if .Values.ingress.enabled -}}
{{- $serviceName := include "kubernetes-dashboard.fullname" . -}}
{{- $servicePort := .Values.service.externalPort -}}
{{- $paths := .Values.ingress.paths -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ template "kubernetes-dashboard.fullname" . }}
  labels:
    {{- include "kubernetes-dashboard.labels" . | nindent 4 }}
    {{- range $key, $value := .Values.ingress.labels }}
    {{ $key }}: {{ $value | quote }}
    {{- end }}
    {{- if .Values.commonLabels }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonLabels "context" $ ) | nindent 4 }}
    {{- end }}
  annotations:
    {{- if .Values.commonAnnotations }}
    {{- include "common.tplvalues.render" ( dict "value" .Values.commonAnnotations "context" $ ) | nindent 4 }}
    {{- end }}
    {{- if not .Values.protocolHttp }}
    # Add https backend protocol support for ingress-nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # Add https backend protocol support for GKE
    service.alpha.kubernetes.io/app-protocols: '{"https":"HTTPS"}'
    {{- end }}
    {{- with .Values.ingress.annotations }}
    {{ toYaml . | nindent 4 }}
    {{- end }}
spec:
  {{- with .Values.ingress.className }}
  ingressClassName: {{ . | quote }}
  {{- end }}
  rules:
  {{- if .Values.ingress.hosts }}
  {{- range $host := .Values.ingress.hosts }}
    - host: {{ $host }}
      http:
        paths:
  {{- if len ($.Values.ingress.customPaths) }}
  {{- "\n" }}{{ tpl (toYaml $.Values.ingress.customPaths | nindent 10) $ }}
  {{- else }}
  {{- range $p := $paths }}
          - path: {{ $p }}
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ $serviceName }}
                port:
                  number: {{ $servicePort }}
  {{- end -}}
  {{- end -}}
  {{- end -}}
  {{- else }}
    - http:
        paths:
  {{- if len ($.Values.ingress.customPaths) }}
  {{- "\n" }}{{ tpl (toYaml $.Values.ingress.customPaths | nindent 10) $ }}
  {{- else }}
  {{- range $p := $paths }}
          - path: {{ $p }}
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ $serviceName }}
                port:
                  number: {{ $servicePort }}
  {{- end -}}
  {{- end -}}
  {{- end -}}
  {{- if .Values.ingress.tls }}
  tls:
{{ toYaml .Values.ingress.tls | nindent 4 }}
  {{- end -}}
{{- end -}}
```

This is what the `values.yaml` has to say about the `ingress` definition:

```sh
cat ../kubernetes-dashboard/values.yaml | grep "^ingress:" -B 1 -A 59
```

which is:

```yml

ingress:
  ## If true, Kubernetes Dashboard Ingress will be created.
  ##
  enabled: false

  ## Kubernetes Dashboard Ingress labels
  # labels:
  #   key: value

  ## Kubernetes Dashboard Ingress annotations
  # annotations:
  #   kubernetes.io/ingress.class: nginx
  #   kubernetes.io/tls-acme: 'true'

  ## If you plan to use TLS backend with enableInsecureLogin set to false
  ## (default), you need to uncomment the below.
  ## If you use ingress-nginx < 0.21.0
  #   nginx.ingress.kubernetes.io/secure-backends: "true"
  ## if you use ingress-nginx >= 0.21.0
  #   nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

  ## Kubernetes Dashboard Ingress Class
  # className: "example-lb"

  ## Kubernetes Dashboard Ingress paths
  ## Both `/` and `/*` are required to work on gce ingress.
  paths:
    - /
  #  - /*

  ## Custom Kubernetes Dashboard Ingress paths. Will override default paths.
  ##
  customPaths: []
  #  - pathType: ImplementationSpecific
  #    backend:
  #      service:
  #        name: ssl-redirect
  #        port:
  #          name: use-annotation
  #  - pathType: ImplementationSpecific
  #    backend:
  #      service:
  #        name: >-
  #          {{ include "kubernetes-dashboard.fullname" . }}
  #        port:
  #          # Don't use string here, use only integer value!
  #          number: 443
  ## Kubernetes Dashboard Ingress hostnames
  ## Must be provided if Ingress is enabled
  ##
  # hosts:
  #   - kubernetes-dashboard.domain.com
  ## Kubernetes Dashboard Ingress TLS configuration
  ## Secrets must be manually created in the namespace
  ##
  # tls:
  #   - secretName: kubernetes-dashboard-tls
  #     hosts:
  #       - kubernetes-dashboard.domain.com

```

The `values.yaml` documentation is a little confusing and the template quite difficult to decipher at once so break it apart to see what is happening here:

- annotations are to be added to the ingress on the condition that `protocolHttp` is not true. Excerpt from ingress template:

  ```yml
  {{- if not .Values.protocolHttp }}
    # Add https backend protocol support for ingress-nginx
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    # Add https backend protocol support for GKE
    service.alpha.kubernetes.io/app-protocols: '{"https":"HTTPS"}'
    {{- end }}
  ```

- the `rules` entries section is basically repeated in the template, one time for all the hosts if any are given or once without `host` property if no hosts are given. Regarding the `paths` to process in both blocks the template will write out all fully specified `customPaths` if any are provided or will iterate over the predefined `paths` if no `customPaths` are given where the `paths` all point to the `dashboard` Service:

  ```yml
  rules:
  {{- if .Values.ingress.hosts }}
  {{- range $host := .Values.ingress.hosts }}
    - host: {{ $host }}
      http:
        paths:
  {{- if len ($.Values.ingress.customPaths) }}
  {{- "\n" }}{{ tpl (toYaml $.Values.ingress.customPaths | nindent 10) $ }}
  {{- else }}
  {{- range $p := $paths }}
          - path: {{ $p }}
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ $serviceName }}
                port:
                  number: {{ $servicePort }}
  {{- end -}}
  {{- end -}}
  {{- end -}}
  {{- else }}
    - http:
        paths:
  {{- if len ($.Values.ingress.customPaths) }}
  {{- "\n" }}{{ tpl (toYaml $.Values.ingress.customPaths | nindent 10) $ }}
  {{- else }}
  {{- range $p := $paths }}
          - path: {{ $p }}
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ $serviceName }}
                port:
                  number: {{ $servicePort }}
  {{- end -}}
  {{- end -}}
  {{- end -}}
  ```

Here is a simple approach to model this ingress where the default rule is being added as would be the case in the original `values.yaml` configuration:

```sh
echo '    ingress:
      dashboard:
        enabled: false
        annotations: |- 
          _HT!{
            {{ if not (index . "$").Values.hull.config.specific.protocolHttp }}
            "nginx.ingress.kubernetes.io/backend-protocol": "HTTPS",
            "service.alpha.kubernetes.io/app-protocols": "{\"https\":\"HTTPS\"}"
            {{ end }}
          }
        rules:
          default:
            http:
              paths:
                root:
                  path: /
                  pathType: ImplementationSpecific
                  backend:
                    service:
                      name: dashboard
                      port:
                        number: _HT*hull.config.specific.externalPort' >> values.yaml
```

If you need to change it at deploy time you could:
- add a `host` to the `default` path
- alter the `path` for the `default` rule for example to match your wanted route
- set the `rule` for `default: null` in which it will not be rendered anymore and create your own `rule`

As a sanity check you should enable the ingress to see it prints out as expected:

```sh
echo 'hull:
  objects:
    ingress:
      dashboard:
        enabled: true' > ../configs/enable-ingress.yaml
```

Check the result:

```sh
helm template -f ../configs/enable-ingress.yaml .
```

and there it is:

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
apiVersion: v1
kind: Service
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
    kubernetes.io/cluster-service: "true"
  name: release-name-kubernetes-dashboard-dashboard
spec:
  ports:
  - name: https
    port: 443
    targetPort: https
  selector:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/name: kubernetes-dashboard
  type: ClusterIP
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: HTTPS
    service.alpha.kubernetes.io/app-protocols: '{"https":"HTTPS"}'
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
  rules:
  - host:
    http:
      paths:
      - backend:
          service:
            name: release-name-kubernetes-dashboard-dashboard
            port:
              number: 443
        path: /
        pathType: ImplementationSpecific
  tls: []
```

Since the `protocolHttp` option has a significant effect on the rendered output also do a quick test if everything is in order:

```sh
echo 'hull:
  config:
    specific:
      protocolHttp: true
  objects:
    ingress:
      dashboard:
        enabled: true' > ../configs/enable-ingress-http.yaml
```

Check the result:

```sh
helm template -f ../configs/enable-ingress-http.yaml .
```

and there it is:

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
apiVersion: v1
kind: Service
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
    kubernetes.io/cluster-service: "true"
  name: release-name-kubernetes-dashboard-dashboard
spec:
  ports:
  - name: http
    port: 443
    targetPort: http
  selector:
    app.kubernetes.io/component: dashboard
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/name: kubernetes-dashboard
  type: ClusterIP
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
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
  rules:
  - host:
    http:
      paths:
      - backend:
          service:
            name: release-name-kubernetes-dashboard-dashboard
            port:
              number: 443
        path: /
        pathType: ImplementationSpecific
  tls: []
```

## Another Wrap Up

So to finish this quickly:

1. That is how your `values.yaml`should look right now:

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
        service:
          dashboard:
            labels:
              ## Enable or disable the kubernetes.io/cluster-service label. Should be disabled for GKE clusters >=1.15.
              ## Otherwise, the addon manager will presume ownership of the service and try to delete it.
              "kubernetes.io/cluster-service": "true"
            type: ClusterIP
            ports:
              http:
                enabled: _HT?(index . "$").Values.hull.config.specific.protocolHttp
                port: _HT*hull.config.specific.externalPort
                targetPort: http
              https:
                enabled: _HT?(not (index . "$").Values.hull.config.specific.protocolHttp)
                port: _HT*hull.config.specific.externalPort
                targetPort: https
        ingress:
          dashboard:
            enabled: false
            annotations: |-
              _HT!{
                {{ if not (index . "$").Values.hull.config.specific.protocolHttp }}
                "nginx.ingress.kubernetes.io/backend-protocol": "HTTPS",
                "service.alpha.kubernetes.io/app-protocols": "{\"https\":\"HTTPS\"}"
                {{ end }}
              }
            rules:
              default:
                http:
                  paths:
                    root:
                      path: /
                      pathType: ImplementationSpecific
                      backend:
                        service:
                          name: dashboard
                          port:
                            number: _HT*hull.config.specific.externalPort
    ```

2. Run the `values.yaml` backup:

    ```sh
    cp values.yaml values.tutorial-part.yaml
    ```

3. Add in the already created objects:

    ```sh
    sed '1,/objects:/d' values.full.yaml > _tmp && cp values.yaml values.full.yaml && cat _tmp >> values.full.yaml && rm _tmp
    ```

Thanks for taking part and hope to see you in the next tutorial part as well!