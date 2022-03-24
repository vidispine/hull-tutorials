
# Summary

To finish this tutorial let's do some comparisons. For this copy over once more the last chart state which is also the final one and make the `values.full.yaml` the `values.yaml`:

```sh
cd ~/kubernetes-dashboard-hull && cp -R 07_advanced_configuration/ 08_summary && cd 08_summary/kubernetes-dashboard-hull && rm values.yaml && cp values.full.yaml values.yaml
```

And here is the final `values.yaml`:

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
    servicemonitor:
      dashboard:
        enabled: false
        endpoints: |-
          _HT![
            {{ if (index . "$").Values.hull.config.specific.protocolHttp }}
            { port: "http" }
            {{ else }}
            { port: "https" }
            {{ end }}
          ]
        selector:
          matchLabels: _HT&dashboard
    networkpolicy:
      dashboard:
        enabled: false
        labels:
          app: _HT!{{- default (index . "$").Chart.Name (index . "$").Values.nameOverride | trunc 63 | trimSuffix "-" -}}
          chart: _HT!{{- printf "%s-%s" (index . "$").Chart.Name (index . "$").Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
          release: _HT!{{ (index . "$").Release.Name }}
          heritage: _HT!{{ (index . "$").Release.Service }}
        podSelector:
          matchLabels: _HT&dashboard
        ingress: |-
          _HT![
            { ports:
              [
                {
                  protocol: TCP,
                  {{ if (index . "$").Values.hull.config.specific.protocolHttp }}
                  port: "http"
                  {{ else }}
                  port: "https"
                  {{ end }}
                }
              ]
            }
          ]
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
            resourceNames: _HT![ {{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "psp") }} ]
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
    configmap:
      settings:
        data:
          _global:
            inline: _HT!{{ toJson (index . "$").Values.hull.config.specific.settings | quote }}
          _pinnedCRD:
            path: files/pinnedCRDs
    secret:
      certs:
        data: {}
      kubernetes-dashboard-csrf:
        data: {}
        staticName: true
      kubernetes-dashboard-key-holder:
        data: {}
        staticName: true
```

It could do with some sorting of object types and brushing up with __lots__ of comments and documentation. The effort not spend on template creation should be put into writing good and extensive documentation of the `values.yaml` - only for reasons of keeping everything shorter no comments were added in this tutorial.

Let's render it finally with everything included:

```sh
helm template .
```

and here it is - the default configuration of `kubernetes-dashboard-hull`:

```yml
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
apiVersion: v1
kind: Secret
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: certs
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-certs
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
kind: Secret
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: kubernetes-dashboard-csrf
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: kubernetes-dashboard-csrf
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
kind: Secret
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: kubernetes-dashboard-key-holder
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: kubernetes-dashboard-key-holder
---
# Source: kubernetes-dashboard/templates/hull.yaml
apiVersion: v1
data:
  _global: '{}'
  _pinnedCRD: '{}'
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: settings
    app.kubernetes.io/instance: release-name
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: kubernetes-dashboard
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 2.5.0
    helm.sh/chart: kubernetes-dashboard-5.2.0
  name: release-name-kubernetes-dashboard-settings
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

## Lines of Code

Assuming you have created that functionally resembles the original `kubernetes-dashboard` Helm chart, let's consider some numbers. 

Here is a Lines of Code analysis performed with the Visual Studio Code LoC Extension for the original `kubernetes-dashboard` charts relevant configuration files (`values.yaml` + templates):

| File | Lines of Code | Comments
| ------------------------------- | ----------------------------------------------------------------| -----------------------------
| `values.yaml` | 149 | 161
| `templates/_helpers.tpl` | 64 | 15
| `templates/_tplvalues.tpl` | 23 | 5
| `templates/clusterrole-metrics.yaml` | 33 | 1
| `templates/clusterrole-readonly.yaml` | 153 | 1
| `templates/clusterrolebinding-metrics.yaml` | 36 | 1
| `templates/clusterrolebinding-readonly.yaml` | 36 | 1
| `templates/configmap.yaml` | 33 | 1
| `templates/deployment.yaml` | 186 | 2
| `templates/ingress.yaml` | 88 | 1
| `templates/networkpolicy.yaml` | 44 | 1
| `templates/pdb.yaml` | 38 | 1
| `templates/psp.yaml` | 81 | 1
| `templates/role.yaml` | 48 | 1
| `templates/rolebinding.yaml` | 36 | 1
| `templates/secret.yaml` | 46 | 1
| `templates/service.yaml` | 58 | 1
| `templates/serviceaccount.yaml` | 28 | 1
| `templates/servicemonitor.yaml` | 40 | 1
| Total | 1220 | 198

Essentially there exist ~1220 lines of configurational code written which need to be maintained by the chart maintainers. The chart consumers need to understand and maintain ~150 lines of configuration in the `values.yaml` but will likely need to check the templates too to understand the inner workings.

Compared to that, the `kubernetes-dashboard-hull`'s `values.yaml` - containing all configuration code - consists of a mere 460 lines of configuration. With only about a third of the original configuration code to maintain, the HULL based chart arguably covers the default structure and logic of the source chart. But still it is relatively unrestricted in the ways you can adapt the configuration to your needs in any given deployment scenario.

Thank you for staying with us and hope you enjoyed the turorial :)



