## Adding ConfigMaps and Secrets

As a reminder, the goal of this tutorial series is to demonstrate how to build Helm charts based on the [HULL library chart](https://github.com/vidispine/hull) by recreating the functionality of the original `nginx-ingress` Helm chart with a HULL based chart from scratch. When you have followed the previous part of this tutorial [on setting up a HULL base chart](https://dev.to/gre9ory/setup-a-helm-chart-based-on-hull-44i6-temp-slug-7198775) you have created a for now unconfigured Helm chart named `ingress-nginx-hull` in the `02_setup` subfolder of your working directory (we assume that's `~` here). You can alternatively download the current chart state [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/02_setup) and continue from there. Also you should have checked out and extracted the `ingress-nginx` Helm chart to `ingress-nginx` in your working directory because examining it will be frequently required:

```bash
$ cd ~
```
```bash
$ ls 02_setup/ingress-nginx*
```

should give you:

```bash
02_setup/ingress-nginx:
CHANGELOG.md  Chart.yaml  OWNERS  README.md  ci  templates  values.yaml

02_setup/ingress-nginx-hull:
Chart.lock  Chart.yaml  charts  templates  values.yaml
```

To continue you should make a copy for our new tutorial Part and leave the Part 2 result untouched:

```bash
$ cp -R 02_setup/ 03_configmaps_and_secrets
```

Ok, now you can change to our freshly created `ingress-nginx-hull` chart directory:

```bash
$ cd 03_configmaps_and_secrets/ingress-nginx-hull
```

### Analyzing the original ConfigMaps

It is time to start moving the objects from the original to our new chart and focus on ConfigMaps and Secrets as a first step. When checking the `nginx-ingress`'s `/templates` folder with 

```bash
$ find ../ingress-nginx/templates -type f -iregex '.*\(configmap\|secret\).*' | sort
```

we can spot a couple of ConfigMaps and Secrets in there:

```yaml
../ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
../ingress-nginx/templates/controller-configmap-addheaders.yaml
../ingress-nginx/templates/controller-configmap-proxyheaders.yaml
../ingress-nginx/templates/controller-configmap-tcp.yaml
../ingress-nginx/templates/controller-configmap-udp.yaml
../ingress-nginx/templates/controller-configmap.yaml
../ingress-nginx/templates/dh-param-secret.yaml
```

Note that file naming of templates does not actually mean anything in terms of what may be contained in them but for our exercise the result above points in the right direction (the first entry being a job and no secret so it is dealt with in a later chapter). In this part we will work on converting 

- /templates/controller-configmap-addheaders.yaml
- /templates/controller-configmap-proxyheaders.yaml
- /templates/controller-configmap-tcp.yaml
- /templates/controller-configmap-udp.yaml
- /templates/controller-configmap.yaml
- /ingress-nginx/templates/dh-param-secret.yaml

in this order.

You can proceed with getting the contents of the `templates/controller-configmap-addheaders.yaml` ConfigMap:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-addheaders.yaml
```
```yaml
{{- if .Values.controller.addHeaders -}}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ include "ingress-nginx.fullname" . }}-custom-add-headers
  namespace: {{ .Release.Namespace }}
data: {{ toYaml .Values.controller.addHeaders | nindent 2 }}
{{- end }}
```

In the top line of `templates/controller-configmap-addheaders.yaml` a field `controller.addHeaders` from the `values.yaml` seems to trigger the creation of the ConfigMap object in the template. You can check what more can be learnt about the `controller.addHeaders` field in `values.yaml` by inspecting the block where 'addHeaders' appears in `values.yaml`:

```bash
$ cat ../ingress-nginx/values.yaml | grep 'addHeaders' -B 10 -A 10
```
```yaml
  config: {}
## Annotations to be added to the controller config configuration configmap
  ##
  configAnnotations: {}
# Will add custom headers before sending traffic to backends according to https://github.com/kubernetes/ingress-nginx/tree/main/docs/examples/customization/custom-headers
  proxySetHeaders: {}
# Will add custom headers before sending response traffic to the client according to: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#add-headers
  addHeaders: {}
# Optionally customize the pod dnsConfig.
  dnsConfig: {}
# Optionally customize the pod hostname.
  hostname: {}
# Optionally change this to ClusterFirstWithHostNet in case you have 'hostNetwork: true'.
  # By default, while using host network, name resolution uses the host's DNS. If you wish nginx-controller
  # to keep resolving names inside the k8s network, use ClusterFirstWithHostNet.
```

So a dictionary is expected for `controller.addHeaders`. [This StackOverflow topic](https://medium.com/r/?url=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F59795596%2Fhelm-optional-nested-variables) highlights that an empty dictionary will evaluate to false when checked with `{{ if … }}` so the top line instruction is now clear: only create this ConfigMap if there is any key-value pair in the `controller.addHeaders` dictionary field!

Next you can see that some labels are generated by an include of  `ingress-nginx.labels` helper function. In classic Helm charts such helper functions are found usually in a `_helpers.tpl` which is also the case here. To get an overview of this function and the function it internally calls you may display the whole file contents because we will need to get back to them frequently:

```bash
$ cat ../ingress-nginx/templates/_helpers.tpl
```
```yaml
{{/* vim: set filetype=mustache: */}}
{{/*
Expand the name of the chart.
*/}}
{{- define "ingress-nginx.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "ingress-nginx.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{/*
Create a default fully qualified app name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "ingress-nginx.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}
{{/*
Create a default fully qualified controller name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "ingress-nginx.controller.fullname" -}}
{{- printf "%s-%s" (include "ingress-nginx.fullname" .) .Values.controller.name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{/*
Construct the path for the publish-service.
By convention this will simply use the <namespace>/<controller-name> to match the name of the
service generated.
Users can provide an override for an explicit service they want bound via `.Values.controller.publishService.pathOverride`
*/}}
{{- define "ingress-nginx.controller.publishServicePath" -}}
{{- $defServiceName := printf "%s/%s" "$(POD_NAMESPACE)" (include "ingress-nginx.controller.fullname" .) -}}
{{- $servicePath := default $defServiceName .Values.controller.publishService.pathOverride }}
{{- print $servicePath | trimSuffix "-" -}}
{{- end -}}
{{/*
Create a default fully qualified default backend name.
We truncate at 63 chars because some Kubernetes name fields are limited to this (by the DNS naming spec).
*/}}
{{- define "ingress-nginx.defaultBackend.fullname" -}}
{{- printf "%s-%s" (include "ingress-nginx.fullname" .) .Values.defaultBackend.name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{/*
Common labels
*/}}
{{- define "ingress-nginx.labels" -}}
helm.sh/chart: {{ include "ingress-nginx.chart" . }}
{{ include "ingress-nginx.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end -}}
{{/*
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
{{- define "ingress-nginx.defaultBackend.serviceAccountName" -}}
{{- if .Values.defaultBackend.serviceAccount.create -}}
    {{ default (printf "%s-backend" (include "ingress-nginx.fullname" .)) .Values.defaultBackend.serviceAccount.name }}
{{- else -}}
    {{ default "default-backend" .Values.defaultBackend.serviceAccount.name }}
{{- end -}}
{{- end -}}
{{/*
Return the appropriate apiGroup for PodSecurityPolicy.
*/}}
{{- define "podSecurityPolicy.apiGroup" -}}
{{- if semverCompare ">=1.14-0" .Capabilities.KubeVersion.GitVersion -}}
{{- print "policy" -}}
{{- else -}}
{{- print "extensions" -}}
{{- end -}}
{{- end -}}
{{/*
Check the ingress controller version tag is at most three versions behind the last release
*/}}
{{- define "isControllerTagValid" -}}
{{- if not (semverCompare ">=0.27.0-0" .Values.controller.image.tag) -}}
{{- fail "Controller container image tag should be 0.27.0 or higher" -}}
{{- end -}}
{{- end -}}
{{/*
IngressClass parameters.
*/}}
{{- define "ingressClass.parameters" -}}
  {{- if .Values.controller.ingressClassResource.parameters -}}
          parameters:
{{ toYaml .Values.controller.ingressClassResource.parameters | indent 4}}
  {{ end }}
{{- end -}}
```

As already demonstrated in Part 1 of this series, all objects do automatically get the standard Kubernetes metadata fields populated with HULL and unique selectors for pods are created automatically. Further metadata can be added at the global, object type or object level. So there is no real further need to work on the standard metadata of our objects, HULL automatically creates metadata information that is equivalent to what happens in: 

```yaml
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ include "ingress-nginx.fullname" . }}-custom-add-headers
```

when you break it down.

With  respect to the object name, in the original templates the object names are generated by combining function `ingress-nginx.fullname` and a static suffix `-custom-add-headers`. This can be considered standard practice for naming objects in Helm charts to add a suffix to `<chart-name>-<release-name>`. In HULL automatic object naming is by default derived in the same manner. Since all Kubernetes objects have a key associated with them in HULL this key name provides the suffix to the `<chart-name>-<release-name>` prefix. To save some typing we will shorten the suffix/name here to `addheaders` from now on. 

Now that the structure of the `templates/controller-configmap-addheaders.yaml` is analysed you can compare it with the `templates/controller-configmap-proxyheaders.yaml` to check for similarities:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-proxyheaders.yaml
```
```yaml
{{- if or .Values.controller.proxySetHeaders .Values.controller.headers -}}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ include "ingress-nginx.fullname" . }}-custom-proxy-headers
  namespace: {{ .Release.Namespace }}
data:
{{- if .Values.controller.proxySetHeaders }}
{{ toYaml .Values.controller.proxySetHeaders | indent 2 }}
{{ else if and .Values.controller.headers (not .Values.controller.proxySetHeaders) }}
{{ toYaml .Values.controller.headers | indent 2 }}
{{- end }}
{{- end }}
```

Apparently there is a lot of similarity between the two ConfigMaps. The only noteworthy difference is that two fields `controller.headers` and `controller.proxySetHeaders` are or-ed together to control the creation of the ConfigMap object. 

A search on `proxySetHeaders`:

```bash
$ find ../ingress-nginx/templates -type f -print | xargs grep "proxySetHeaders"
```

yields that it is amongst others being referenced in `NOTES.txt`:

```yaml
../ingress-nginx/templates/NOTES.txt:######            It has been renamed to `controller.proxySetHeaders`.      #####
../ingress-nginx/templates/controller-configmap.yaml:{{- if or .Values.controller.proxySetHeaders .Values.controller.headers }}
../ingress-nginx/templates/controller-configmap-proxyheaders.yaml:{{- if or .Values.controller.proxySetHeaders .Values.controller.headers -}}
../ingress-nginx/templates/controller-configmap-proxyheaders.yaml:{{- if .Values.controller.proxySetHeaders }}
../ingress-nginx/templates/controller-configmap-proxyheaders.yaml:{{ toYaml .Values.controller.proxySetHeaders | indent 2 }}
../ingress-nginx/templates/controller-configmap-proxyheaders.yaml:{{ else if and .Values.controller.headers (not .Values.controller.proxySetHeaders) }}
```

If you dig deeper with:

```bash
$ cat ../ingress-nginx/templates/NOTES.txt | grep 'proxySetHeaders' -B 10 -A 10
```
```yaml
    name: example-tls
    namespace: foo
  data:
    tls.crt: <base64 encoded cert>
    tls.key: <base64 encoded key>
  type: kubernetes.io/tls
{{- if .Values.controller.headers }}
#################################################################################
######   WARNING: `controller.headers` has been deprecated!                 #####
######            It has been renamed to `controller.proxySetHeaders`.      #####
#################################################################################
{{- end }}
```

it becomes clear that `controller.proxySetHeaders` replaced the `controller.headers` field at some point so we have a case of downward compatibility: if `controller.headers` is set use this field, otherwise check `controller.proxySetHeaders`. In `ingress-nginx-hull` simply one source of truth will be used.

To summarize the inspection sofar of the two ConfigMaps we have the same structure in both files with the slight exception of the downward compatibility handling in the `controller.proxySetHeaders` case. 

### Creating ConfigMaps with HULL

Time get down to business by creating the first ConfigMaps in the HULL charts `values.yaml`! 

Copy or type in: 

```bash
$ echo 'hull:
  objects:
    configmap:
      addheaders:
        enabled: true
        data: {}
      proxyheaders:
        enabled: true
        data: {}
' > values.yaml
```

to create the two empty ConfigMaps in `values.yaml` and check the output with:

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
apiVersion: v1
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: addheaders
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-addheaders
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: proxyheaders
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-proxyheaders
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

There are the two new ConfigMaps in the output but right now they are just empty. In the original chart the logic forced a conditional rendering of the objects only when they have entries in the `data:` section defined. 

At this point there two ways ways to model this behavior in our new chart. One way is the explicit way where we set `enabled: false` on both ConfigMaps to suppress rendering them by default and demand to manually set `enabled: true` whenever any entries in the respective `data: {}` dictionaries are created. That would work but the downside is that when this is not documented well enabling the rendering may be forgotten and the configuration not producing the intended result. The other way is to model the original automatism to just craete the object when it has actual data. But how can such a dynamic decision be made when we don't use the templating capabilities from the files in the `/templates` folder?  HULL transformations are the answer and we will use them to model the dynamic behavior now for simplifying user experience in sake of some explicitness.

### Using HULL transformations

In the regular Helm workflow any templating or dynamic behavior is not allowed within `values.yaml` itself, only in the `/templates` folder. Transformations in HULL provide the possibility to dynamically adjust property values within `values.yaml` itself however. How is this done?

Since within HULL the whole `values.yaml` tree is traversed it enables the possibility to inject templating again into property value calculation by processing keywords that signal the use of a dynamic function (=transformation) and its parameters. The return value of the transformation then replaces the property value where the transformation was called. A caveat is that the specification and use of those functions must not invalidate the YAML and JSON schema of the `values.yaml`, otherwise Helm would not be happy about processing the `values.yaml`. For this reason the transformations themselves are wrappable in form of an string, dictionary or array. The HULL transformation trigger `_HULL_TRANSFORMATION_`signals use of a transformation to determine the actual value to be used. More information about transformations and how to specify and use them can be found in the [HULL docs on transformations](https://github.com/vidispine/hull/blob/main/hull/doc/transformations.md).

#### The `hull.util.transformation.bool` function

The `hull.util.transformation.bool` function is a first simple transformation example albeit a useful one to determine whether a particular object should be rendered or not at deploy time. For all HULL objects this is controlled via the boolean `enabled` switch, you can set it for each main object defined through HULL. By binding the `enabled` property's value to a `hull.util.transformation.bool` function you can achieve the desired behavior discussed beforehand: use a transformation on the `enabled` properties to toggle them to `true` whenever the respective `data` section has entries and `false` otherwise. 

Now start again from scratch to add the transformation:

```bash
$ echo 'hull:
  objects:
    configmap:
      addheaders:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.objects.configmap.addheaders.data>>>"
        data: {}
      proxyheaders:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.objects.configmap.proxyheaders.data>>>"
        data: {}' > values.yaml
```

Rendering this:

```bash
$ helm template .
```

produces:

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

and there are no ConfigMaps in the output. This is of course actually correct since no `data` entries were given so far!

To add some example `addheaders` and `proxyheaders` to the configuration it is time to bring in another layer of configuration that helps in performing our tests of the functionality and simulate system specific settings in addition to the `values.yaml` defaults. An 'external `values.yaml`' is required to vary the configuration on top of the defaults and see the effect on rendering outputs. Before any HULL related processing takes place, the contents of this system/test file are simply merged entry by entry with the `values.yaml` structure when applied with `-f <file-path>` to any `helm` command. Note that any data that is added via the `-f` switch has precedence over the values.yaml defaults (which is what you'd expect anyways). 

Create a new `../configs` folder now to hold such 'external configurations' to be applied on top of our default `values.yaml`:

```bash
$ mkdir ../configs
```

Add a first test configuration like this by adding two headers to `addheaders` ConfigMap:

```bash
$ echo 'hull:
  objects:
    configmap:
      addheaders:
        data: 
          header1: 
            inline: This is a first header
          header2: 
            inline: This is a second header
' > ../configs/two-add-headers.yaml
```

Rerender now by overlaying the additional configuration with our defaults and check the result:

```bash
$ helm template -f ../configs/two-add-headers.yaml .
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
apiVersion: v1
data:
  header1: This is a first header
  header2: This is a second header
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: addheaders
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-addheaders
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
```

It worked - you now have dynamically created the `addheaders` ConfigMap just by adding data to it! Try it the other way around to verify it was not by accident by creating a second 'external configuration' that enables the `proxyheaders` instead and disables the `addheaders`: 

```bash
$ echo 'hull:
  objects:
    configmap:
      proxyheaders:
        data: 
          header1: 
            inline: This is a first header
          header2: 
            inline: This is a second header
' > ../configs/two-proxy-headers.yaml
```

and apply:

```bash
$ helm template -f ../configs/two-proxy-headers.yaml .
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
apiVersion: v1
data:
  header1: This is a first header
  header2: This is a second header
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: proxyheaders
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-proxyheaders
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
```

Now there is the `proxyheaders` ConfigMap and no `addheaders` ConfigMap which should suffice to prove that the transformation actually works!

#### Specyfiying ConfigMap and Secret content inside `values.yaml`

You likely noticed that within the 'external configuration' files `data` section the actual values for `data` keys were directly given via the `inline` property. While it is possible and in some cases better (see further below) to import ConfigMap and Secret `data` values from external files via the `files` property, it is handy to be able to specify shorter contents right in place where the ConfigMap is defined. This of course also works within `values.yaml` itself and not only in overlays as in the examples.

Time to move on to the next ConfigMap pair, the `/templates/controller-configmap-tcp.yaml` and `/templates/controller-configmap-udp.yaml`:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-tcp.yaml
```
```yaml
{{- if .Values.tcp -}}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
{{- if .Values.controller.tcp.annotations }}
  annotations: {{ toYaml .Values.controller.tcp.annotations | nindent 4 }}
{{- end }}
  name: {{ include "ingress-nginx.fullname" . }}-tcp
  namespace: {{ .Release.Namespace }}
data: {{ tpl (toYaml .Values.tcp) . | nindent 2 }}
{{- end }}
```
and:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-udp.yaml
```
```yaml
{{- if .Values.udp -}}
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
{{- if .Values.controller.udp.annotations }}
  annotations: {{ toYaml .Values.controller.udp.annotations | nindent 4 }}
{{- end }}
  name: {{ include "ingress-nginx.fullname" . }}-udp
  namespace: {{ .Release.Namespace }}
data: {{ tpl (toYaml .Values.udp) . | nindent 2 }}
{{- end }}
```

Both are also very alike and look similar to the `addheaders` case in structure. Using the same approach of 'if there is something in `data:` we render the ConfigMap, otherwise not' seems reasonable but investigation of further usages shows us that the `tcp` and `udp` values are used in multiple places in the original chart:


```bash
$ find ../ingress-nginx/templates -type f -print | xargs grep ".Values.tcp"
```
```yaml
../ingress-nginx/templates/controller-configmap-tcp.yaml:{{- if .Values.tcp -}}
../ingress-nginx/templates/controller-configmap-tcp.yaml:data: {{ tpl (toYaml .Values.tcp) . | nindent 2 }}
../ingress-nginx/templates/controller-daemonset.yaml:          {{- if .Values.tcp }}
../ingress-nginx/templates/controller-daemonset.yaml:          {{- range $key, $value := .Values.tcp }}
../ingress-nginx/templates/controller-deployment.yaml:          {{- if .Values.tcp }}
../ingress-nginx/templates/controller-deployment.yaml:          {{- range $key, $value := .Values.tcp }}
../ingress-nginx/templates/controller-psp.yaml:{{- range $key, $value := .Values.tcp }}
../ingress-nginx/templates/controller-service-internal.yaml:  {{- range $key, $value := .Values.tcp }}
../ingress-nginx/templates/controller-service.yaml:  {{- range $key, $value := .Values.tcp }}
```

```bash
$ find ../ingress-nginx/templates -type f -print | xargs grep ".Values.udp"
```
```yaml
../ingress-nginx/templates/controller-configmap-udp.yaml:{{- if .Values.udp -}}
../ingress-nginx/templates/controller-configmap-udp.yaml:data: {{ tpl (toYaml .Values.udp) . | nindent 2 }}
../ingress-nginx/templates/controller-daemonset.yaml:          {{- if .Values.udp }}
../ingress-nginx/templates/controller-daemonset.yaml:          {{- range $key, $value := .Values.udp }}
../ingress-nginx/templates/controller-deployment.yaml:          {{- if .Values.udp }}
../ingress-nginx/templates/controller-deployment.yaml:          {{- range $key, $value := .Values.udp }}
../ingress-nginx/templates/controller-psp.yaml:{{- range $key, $value := .Values.udp }}
../ingress-nginx/templates/controller-service-internal.yaml:  {{- range $key, $value := .Values.udp }}
../ingress-nginx/templates/controller-service.yaml:  {{- range $key, $value := .Values.udp }}
```

This time it seems desirable keep the data outside of the ConfigMap object itself because `tcp` and `udp` mappings ideally are defined in a 'single source of truth' fashion as in the original chart. To cover the dynamic nature of the `tcp` and `udp` entries (it is not known if and how many entries may be provided by the user at deploy time) the more static HULL key value pair based `data: ` section approach does not fit and it is needed to create the whole `data:` section within a transformation. This approach has some added complexity but it helps us to maintain a single source of truth for crucial values to simplify configuration for the user. Of course this reduces the straight-forwardness and transparency of the HULL approach when it comes to configuring object properties but for some configurations it still makes sense to abstract them. In other words: with HULL you do not have to literally 'spell out' each projection between `values.yaml` and the rendered objects by putting it into the `template` files but whenever an individual projection improves user experience it should be applied.

#### Entering the `hull.util.transformation.tpl` function

There are some convenience transformations as the `hull.util.transformation.bool` transformation within HULL but the most powerful is the HULL `hull.util.transformation.tpl` function. It executes the Helm `tpl` function on a given string providing full flexibility of what to render. Using the `hull.util.transformation.tpl` function you can return different datatypes that are expected for the property's value:

- string
- int
- bool
- array/list
- dictionary/map

An important thing to keep in mind when producing arrays and dictionaries with the HULL `hull.util.transformation.tpl` transformation is to use [`flow` style YAML](https://cloudslang-docs.readthedocs.io/en/latest/overview/yaml_overview.html#indentation-scoping) to define the transformation result. This style prevents indendation problems as with the `block` style YAML that is more typically used. Check the next example on how flow style YAML looks like (it is more similar to JSON).

Creating the `data:` sections for the `tcp` and `udp` ConfigMaps will now be done by referencing to a 'single source of truth' in the `values.yaml`. That 'source of truth' for the `tcp` and `udp` mappings is best defined in the `hull.config.specific` configuration section. This section is specifically intended for this purpose of adding central configuration data that is either singled out for convenient documentation and configuration or being used in multiple places within the chart. You can think of this section resembling the function of the `values.yaml` itself within the classic Helm chart approach. 

To create the `tcp` and `udp` sections that will centrally hold the data we insert into the existing `values.yaml` the `config` block:

  ```bash
  $ sed '/^hull/r'<(
      echo "  config:"
      echo "    specific:"
      echo "      controller:"
      echo "        configMaps:"
      echo "          tcp:"
      echo "            mappings: {}"
      echo "          udp:"
      echo "            mappings: {}"
    ) -i -- values.yaml
  ```
  
Time to add the `tpl` transformation to create the ConfigMaps `data:` fields from the now created and referenced sections, the enabled property will be handled by the `hull.util.transformation.bool` transformation as previously. Note that the `_HULL_TRANSFORMATION_` usage here comes in form of a dictionary declaration since the data type for the `data` section is dictionary:

```bash
$ echo '      tcp:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.tcp.mappings>>>" 
        data: 
          _HULL_TRANSFORMATION_:
            NAME: hull.util.transformation.tpl
            CONTENT: "
              {
              {{ range $key,$value :=  (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.tcp.mappings }}
              {{ $key }}: { inline: {{ $value }} },
              {{ end }}
              }"
      udp:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.udp.mappings>>>" 
        data:
          _HULL_TRANSFORMATION_:
            NAME: hull.util.transformation.tpl
            CONTENT: "
              {
              {{ range $key,$value :=  (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.udp.mappings }}
              {{ $key }}: { inline: {{ $value }} },
              {{ end }}
              }"' >> values.yaml
```

To verify it is working create an external file again to test the functionality:

```bash
$ echo 'hull:
  config:
    specific:
      controller:
        configMaps:
          tcp:
            mappings: 
              "12345": some_tcp_service
          udp:
            mappings: 
              "67890": some_udp_service
' > ../configs/tcp-udp-mappings.yaml
```

Apply and check:

```bash
$ helm template -f ../configs/tcp-udp-mappings.yaml .
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
apiVersion: v1
data:
  "12345": some_tcp_service
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: tcp
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-tcp
---
# Source: ingress-nginx-hull/templates/hull.yaml
apiVersion: v1
data:
  "67890": some_udp_service
kind: ConfigMap
metadata:
  annotations: {}
  labels:
    app.kubernetes.io/component: udp
    app.kubernetes.io/instance: RELEASE-NAME
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: ingress-nginx-hull
    app.kubernetes.io/part-of: undefined
    app.kubernetes.io/version: 1.0.4
    helm.sh/chart: ingress-nginx-hull-4.0.6
  name: release-name-ingress-nginx-hull-udp
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

All done here. Only one ConfigMap left on the list, the `templates/controller-configmap.yaml`. This is what it looks like:

```bash
$ cat ../ingress-nginx/templates/controller-configmap.yaml
```
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
{{- if .Values.controller.configAnnotations }}
  annotations: {{ toYaml .Values.controller.configAnnotations | nindent 4 }}
{{- end }}
  name: {{ include "ingress-nginx.controller.fullname" . }}
  namespace: {{ .Release.Namespace }}
data:
  allow-snippet-annotations: "{{ .Values.controller.allowSnippetAnnotations }}"
{{- if .Values.controller.addHeaders }}
  add-headers: {{ .Release.Namespace }}/{{ include "ingress-nginx.fullname" . }}-custom-add-headers
{{- end }}
{{- if or .Values.controller.proxySetHeaders .Values.controller.headers }}
  proxy-set-headers: {{ .Release.Namespace }}/{{ include "ingress-nginx.fullname" . }}-custom-proxy-headers
{{- end }}
{{- if .Values.dhParam }}
  ssl-dh-param: {{ printf "%s/%s" .Release.Namespace (include "ingress-nginx.controller.fullname" .) }}
{{- end }}
{{- range $key, $value := .Values.controller.config }}
    {{ $key | nindent 2 }}: {{ $value | quote }}
{{- end }}
```

Apparently this file is the central configuration file for the nginx ingress controller container itself. In it there are four `data:` keys that need to be dealt with and since the application likely expects them to exist exactly like that their key names cannot be changed in any way:
1. the `add-headers` key and value is set dependent on existence of `controller.addHeaders` entries. The logic seems pretty clear: whenever a `controller.addHeaders` dictionary is defined, an `add-headers` key for it is created in `templates/controller-configmap.yaml` and the value points to the additionally created ConfigMap object defined in `templates/controller-configmap-addheaders.yaml` where the `controller.addHeaders` key-value pairs are stored.
2. the `proxy-set-headers` key and value does the same for `proxyheaders` as what the `add-headers` key does for the `addheaders` ConfigMap.
3. the `allow-snippet-annotations` key whose value comes directly from `controller.allowSnippetAnnotations`. It is a simple boolean value as you can see here:

    ```bash
    $ cat ../ingress-nginx/values.yaml | grep 'allowSnippetAnnotations' -B 10 -A 10
    ```

    giving:

    ```yaml
      # Defaults to false
      watchIngressWithoutClass: false
    # Process IngressClass per name (additionally as per spec.controller)
      ingressClassByName: false
    # This configuration defines if Ingress Controller should allow users to set
      # their own *-snippet annotations, otherwise this is forbidden / dropped
      # when users add those annotations.
      # Global snippets in ConfigMap are still respected
      allowSnippetAnnotations: true
    # Required for use with CNI based kubernetes installations 
(such as ones set up by kubeadm),
      # since CNI and hostport don't mix yet. Can be deprecated once https://github.com/kubernetes/kubernetes/issues/23920
      # is merged
      hostNetwork: false
    ## Use host ports 80 and 443
      ## Disabled by default
      ##
      hostPort:
    ```
4. the `ssl-dh-param` key which is dependent on `dhParam` being set. The `dhparam` Secret is checked out in a second, for now just treat the 'if data exists create object' condition same as with the ConfigMaps.

Any other configuration keys and their values can just be added directly to the ConfigMap via HULL standard means in any system-specific `values.yaml`. Hence we don't need to consider the

    ```yaml
    {{- range $key, $value := .Values.controller.config }}
        {{ $key | nindent 2 }}: {{ $value | quote }}
    {{- end }}
    ```

block for explicit modelling further.

#### Importing ConfigMap and Secret `data` via external files

To explore the options that come with HULL this time you will import the `add-headers` key of the `controller` ConfigMap not via the handy `inline` directive but load it from an external file via the `path` directive. It is really simple, just provide a relative path to a file and it will be rendered as a ConfigMap (or Secret) `data:` entry and you don't have to care about anything else such as Base64 encoding for Secrets.
 
As you can see the `add-headers` value is the concatenation of the current namespace with the `addheaders` ConfigMap name. Dynamically computed content of ConfigMap or Secret entries due to usage of template expressions is well suited for supplying it from external files. Especially if you'd have large configuration file like content with many templating expressions it is much more cleaner to provide the file as external content to not clutter the `values.yaml` unneccesarilly.

In this case the content of the `add-headers` key is pretty limited so you would normally also provide it simply `inline`. Using an external file here for the `add-headers` key allows to directly highlight the difference to the `inline` handling of the `proxy-set-headers` key via HULL because functionally the same is happening. As a Bonus you can compare the HULL transformation syntax to the regular templating expression syntax as they are usable within an externally imported file.

Now on to add the `controller` ConfigMap and provide the `add-headers` content via an external file from the `/files` folder and the rest via `inline:` instructions as before. First create the `files` folder and the write the updated `values.yaml` content:

```bash
$ mkdir files
```
```bash
$ echo '{{ .Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $ "COMPONENT" "addheaders") }}' > files/add-headers
```

Create the full ConfigMap definition for the `controller` as discussed:

```bash
$ echo '      controller:
        data:
          add-headers:
            enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
              (index . \"PARENT\").Values.hull.objects.configmap.addheaders.data>>>"
            path: files/add-headers
          set-proxy-headers:
            enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
              (index . \"PARENT\").Values.hull.objects.configmap.proxyheaders.data>>>"
            inline: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.tpl>>><<<CONTENT=
              {{ (index . \"PARENT\").Release.Namespace }}/{{ include \"hull.metadata.fullname\" (dict \"PARENT_CONTEXT\" (index . \"PARENT\") \"COMPONENT\" \"proxyheaders\") }}>>>"
          allow-snippet-annotations:
            inline: "true"
          dhparam.pem:
            enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
              (index (index . \"PARENT\").Values.hull.objects.secret.dhparam.data \"dhparam.pem\").inline>>>"
            inline: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.tpl>>><<<CONTENT=
              {{ (index (index . \"PARENT\").Values.hull.objects.secret.dhparam.data \"dhparam.pem\").inline }}>>>"' >> values.yaml
```

A comparison of the `inline` transformation of `set-proxy-headers` with the regular Go templating expression in the `path` imported `add-headers` yields the syntactical differences you need to take care of when using the HULL transformations inside the `values.yaml`:

```yaml
{{ .Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $ "COMPONENT" "addheaders") }}
```
versus

```yaml
"_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.tpl>>><<<CONTENT=
              {{ (index . \"PARENT\").Release.Namespace }}/{{ include \"hull.metadata.fullname\" (dict \"PARENT_CONTEXT\" (index . \"PARENT\") \"COMPONENT\" \"proxyheaders\") }}>>>"
```

These are the key differences to notice:
- the HULL `hull.util.transformation.tpl` function is a string that starts with the signaling `_HULL_TRANFORMATION_` prefix, the mandatory `NAME` of the function and the particular parameters required for calling the function (here the `CONTENT` for the Helm `tpl` call)
- `<<<` and `>>>` signal the start and end of parameters to a `_HULL_TRANFORMATION_`. These character sequences are currently forbidden within any parameter values!
- access to the Helm charts global context is done via the expression `(index . \"PARENT\")` instead of just `$`. This is a technical implication. If you require multiple accesses to the context you can however simply put something like 

    ```yaml
    {{ $p := (index . \"PARENT\") }}
    ```
    at the top of your `tpl` block and thus use `$p` instead of `(index . \"PARENT\")` for convenience. 
- additionally within the CONTENT block all `"` 's need to be escaped like `\"`. 

But basically that is all that needs to be considered: you should get similar power of dynamically accessing keys in `values.yaml` and templating logic with the HULL transformation as you would with 'regular' way of using Go templating expressions in the `/templates` folder.

#### Specifying Secrets with HULL

After this excourse, there is one Secret referenced here at key `ssl-dh-param` which we need to port over before we can test our chart again and that is the `templates/dh-param-secret.yaml`:

```bash
$ cat ~/ingress-nginx/templates/dh-param-secret.yaml
```
```yaml
{{- with .Values.dhParam -}}
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "ingress-nginx.controller.fullname" $ }}
  labels:
    {{- include "ingress-nginx.labels" $ | nindent 4 }}
data:
  dhparam.pem: {{ . }}
{{- end }}
```

A single `data:` key `dhparam.pem` is created along with the Secret itself in case `dhParam` is set in `values.yaml`. Check the `values.yaml` for the definition of `dhParam`:

```bash
$ cat ../ingress-nginx/values.yaml | grep 'dhParam' -B 10 -A 10
```
```yaml
# UDP service key:value pairs
# Ref: https://github.com/kubernetes/ingress-nginx/blob/main/docs/user-guide/exposing-tcp-udp-services.md
##
udp: {}
#  53: "kube-system/kube-dns:53"
# A base64ed Diffie-Hellman parameter
# This can be generated with: openssl dhparam 4096 2> /dev/null | base64
# Ref: https://github.com/kubernetes/ingress-nginx/tree/main/docs/examples/customization/ssl-dh-param
dhParam:
```

The hit for `dhParam` in the `values.yaml` does not directly indicate the type of the value but the description points to a simple string. 

Luckily creating secrets with HULL is exactly the same process as creating ConfigMaps, all required Base64 encoding is done automatically under the hood even when using the `inline` function. It's easy then to add the `dhparam` Secret in the same manner as the ConfigMaps before:

```bash
$ echo '    secret:
      dhparam:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index (index . \"PARENT\").Values.hull.objects.secret.dhparam.data \"dhparam.pem\").inline>>>"
        data: 
          dhparam.pem: ' >> values.yaml
```

At this point feel free to play around with the already created example files in the `config` folder and check if you get the expected output rendered. For example, seeing the `add-headers: default/release-name-ingress-nginx-hull-addheaders` key value pair in the `controller` ConfigMap appear when templating with `-f ../configs/two-add-headers.yaml`. Hopefully everything behaves as it is supposed to!

## Wrap up

This concludes the subject of ConfigMaps and Secrets. Your `values.yaml` should at this stage look like this:

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
  objects:
    configmap:
      addheaders:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.objects.configmap.addheaders.data>>>"
        data: {}
      proxyheaders:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.objects.configmap.proxyheaders.data>>>"
        data: {}
      tcp:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.tcp.mappings>>>"
        data:
          _HULL_TRANSFORMATION_:
            NAME: hull.util.transformation.tpl
            CONTENT: "
              {
              {{ range $key,$value :=  (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.tcp.mappings }}
              {{ $key }}: { inline: {{ $value }} },
              {{ end }}
              }"
      udp:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.udp.mappings>>>"
        data:
          _HULL_TRANSFORMATION_:
            NAME: hull.util.transformation.tpl
            CONTENT: "
              {
              {{ range $key,$value :=  (index . \"PARENT\").Values.hull.config.specific.controller.configMaps.udp.mappings }}
              {{ $key }}: { inline: {{ $value }} },
              {{ end }}
              }"
      controller:
        data:
          add-headers:
            enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
              (index . \"PARENT\").Values.hull.objects.configmap.addheaders.data>>>"
            path: files/add-headers
          set-proxy-headers:
            enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
              (index . \"PARENT\").Values.hull.objects.configmap.proxyheaders.data>>>"
            inline: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.tpl>>><<<CONTENT=
              {{ (index . \"PARENT\").Release.Namespace }}/{{ include \"hull.metadata.fullname\" (dict \"PARENT_CONTEXT\" (index . \"PARENT\") \"COMPONENT\" \"proxyheaders\") }}>>>"
          allow-snippet-annotations:
            inline: "true"
          dhparam.pem:
            enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
              (index (index . \"PARENT\").Values.hull.objects.secret.dhparam.data \"dhparam.pem\").inline>>>"
            inline: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.tpl>>><<<CONTENT=
              {{ (index (index . \"PARENT\").Values.hull.objects.secret.dhparam.data \"dhparam.pem\").inline }}>>>"
    secret:
      dhparam:
        enabled: "_HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index (index . \"PARENT\").Values.hull.objects.secret.dhparam.data \"dhparam.pem\").inline>>>"
        data:
          dhparam.pem:
```

Still pretty compact for what ground was covered in terms of creatable objects and dynamic functionality.

Now is the right time to make a copy of the current `values.yaml` as `values_intermediate.yaml` in the charts folder:

```bash
$ cp values.yaml values_intermediate.yaml
```

This file is to be updated with the additional content from each upcoming tutorial part and in the end will be the final complete `values.yaml` for the chart which is basically the outcome of the whole tutorial series. By backing up your progress this way the 'working' `values.yaml` can be reset or modified for each tutorial part to limit the output to look at (since we do not require all objects that were created so far each time) but only those in focus of a tutorial part.

Otherwise that is the end of this tutorial part, thanks for reading and hopefully you found it useful and/or interesting!