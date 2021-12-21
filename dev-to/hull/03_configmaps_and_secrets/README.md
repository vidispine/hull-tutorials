## Preparation

As a reminder, the goal of this tutorial series is to demonstrate how to build Helm charts based on the [HULL library chart](https://github.com/vidispine/hull) by recreating the functionality of the original `ingress-nginx` Helm chart with a HULL based chart from scratch. When you have followed the previous part of this tutorial [on setting up a HULL base chart](https://dev.to/gre9ory/setup-a-helm-chart-based-on-hull-44i6-temp-slug-7198775) you have created a for now unconfigured Helm chart named `ingress-nginx-hull` in the `02_setup` subfolder of your working directory (we assume that's `~` here). You can alternatively download the current chart state [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/02_setup) and continue from there. Also you should have checked out and extracted the `ingress-nginx` Helm chart to `ingress-nginx` in your working directory because examining it will be frequently required:

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

To continue you should make a copy for our new tutorial Part and leave the current state result untouched:

```bash
$ cp -R 02_setup/ 03_configmaps_and_secrets
```

Ok, now you can change to our freshly created `ingress-nginx-hull` chart directory:

```bash
$ cd 03_configmaps_and_secrets/ingress-nginx-hull
```

## Analyzing the existing ConfigMaps

It is time to start moving the objects from the original to our new chart and focus on ConfigMaps and Secrets as a first step. When checking the `nginx-ingress`'s `templates` folder with 

```bash
$ find ../ingress-nginx/templates -type f -iregex '.*\(configmap\|secret\).*' | sort
```

you can spot a couple of ConfigMaps and Secrets in there:

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

- `templates/controller-configmap-addheaders.yaml`
- `templates/controller-configmap-proxyheaders.yaml`
- `templates/controller-configmap-tcp.yaml`
- `templates/controller-configmap-udp.yaml`
- `templates/controller-configmap.yaml`
- `templates/dh-param-secret.yaml`

basically in this order.

You can proceed with getting the contents of the `templates/controller-configmap-addheaders.yaml` ConfigMap:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-addheaders.yaml
```

shows this:

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

Now in the top line of `templates/controller-configmap-addheaders.yaml` a field `controller.addHeaders` from the `values.yaml` seems to trigger the creation of the ConfigMap object in the template. You can check what more can be learnt about the `controller.addHeaders` field in `values.yaml` by inspecting the block where 'addHeaders' appears in `values.yaml`:

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

As already demonstrated in the previous part of this tutorial series, all objects do automatically get the standard Kubernetes metadata fields populated with HULL and unique selectors for pods are created automatically. Further metadata can be added at the global, object type or object level. So there is no real further need to work on the standard metadata of our objects, HULL automatically creates metadata information that is equivalent to what happens in: 

```yaml
metadata:
  labels:
    {{- include "ingress-nginx.labels" . | nindent 4 }}
    app.kubernetes.io/component: controller
  name: {{ include "ingress-nginx.fullname" . }}-custom-add-headers
  namespace: {{ .Release.Namespace }}
```

when you break it down because `ingress-nginx.labels`:

```yaml
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
``` 

and `ingress-nginx.selectorLabels`:

```yaml
{{/*
Selector labels
*/}}
{{- define "ingress-nginx.selectorLabels" -}}
app.kubernetes.io/name: {{ include "ingress-nginx.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end -}}
```

in summary populate and add metadatafields 

- `helm.sh/chart`
- `app.kubernetes.io/version`
- `app.kubernetes.io/managed-by`
- `app.kubernetes.io/name`
- `app.kubernetes.io/instance` 

all of which HULL automatically adds to each object created via it so you don't need to explicitly set them for any object.

With  respect to the object name, in the original templates the object names are generated by combining function `ingress-nginx.fullname` and a static suffix `-custom-add-headers`. This can be considered standard practice for naming objects in Helm charts to add a suffix to `<chart-name>-<release-name>`. In HULL automatic object naming is by default derived in the same manner. Since all Kubernetes objects have a key associated with them in HULL this key name provides the suffix to the `<chart-name>-<release-name>` prefix. To save some typing we will shorten the suffix/name here to `addheaders` from now on. 

Now that the structure of the `templates/controller-configmap-addheaders.yaml` is analysed you can compare it with the `templates/controller-configmap-proxyheaders.yaml` to check for similarities:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-proxyheaders.yaml
```

yields:

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

and a diff:

```bash
$ diff ../ingress-nginx/templates/controller-configmap-addheaders.yaml ../ingress-nginx/templates/controller-configmap-proxyheaders.yaml
```

reveals minimal discrepancies between the two ConfigMaps:

```diff
1c1
< {{- if .Values.controller.addHeaders -}}
---
> {{- if or .Values.controller.proxySetHeaders .Values.controller.headers -}}
8c8
<   name: {{ include "ingress-nginx.fullname" . }}-custom-add-headers
---
>   name: {{ include "ingress-nginx.fullname" . }}-custom-proxy-headers
10c10,15
< data: {{ toYaml .Values.controller.addHeaders | nindent 2 }}
---
> data:
> {{- if .Values.controller.proxySetHeaders }}
> {{ toYaml .Values.controller.proxySetHeaders | indent 2 }}
> {{ else if and .Values.controller.headers (not .Values.controller.proxySetHeaders) }}
> {{ toYaml .Values.controller.headers | indent 2 }}
> {{- end }}
```
Apparently there is a lot of similarity between the two ConfigMaps. The only noteworthy difference is that two fields `controller.headers` and `controller.proxySetHeaders` are or-ed together to control the creation of the ConfigMap object and data structure. 

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

## Creating ConfigMaps with HULL

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

At this point there two ways ways to model this behavior in our new chart. One way is the explicit way where we set `enabled: false` on both ConfigMaps to suppress rendering them by default and demand from the configurator to manually set `enabled: true` whenever any entries in the respective `data: {}` dictionaries are created. That would work but the downside is that when this is not documented properly and clearly, enabling the ConfigMap rendering may be forgotten and the configuration is not producing the intended result. The alternative is to attempt to closely model the original automatism to just create the object when it has actual data. But how can such a dynamic decision be made when you can't use the templating capabilities from the files in the `templates` folder?  

You actually can. [HULL transformations](https://github.com/vidispine/hull/blob/main/hull/doc/transformations.md) provide the answer to this problem and they will be used wherever the need arises to model dynamic behavior for improving user experience.

## Using HULL transformations

In the regular Helm workflow any templating expressions or other dynamic behavior is not tolerated within the `values.yaml` itself since it must at all times conform to the YAML specification. Templating expressions are only allowed and evaluated in the `templates` folder. 

The focus and main goal of creating HULL was to get rid of the need to model each simple 1:1 projection from `values.yaml` properties to Kubernetes object properties by providing direct access to all Kubernetes objects properties instead. This is solely achieved by configuration of the `values.yaml` and without any custom templates. 

Transformations in HULL put back the possibility to use template expressions for more complex projections which are driven from the `values.yaml` alone. But how is this done when Helm prohibits any templating expression ins the `values.yaml`?

Since within HULL the whole `values.yaml` tree is traversed internally it enables the possibility to inject templating again into property value calculation by processing keywords that signal the use of a dynamic function (=__transformation__) and its parameters. The return value of the transformation then replaces the property value where the transformation was called. A caveat is that the specification and use of those functions must not invalidate the YAML (and JSON schema) of the `values.yaml`, otherwise Helm would not be happy about processing the `values.yaml`. For this reason the transformations themselves are wrapped in form of strings, dictionaries or arrays. The HULL transformation triggers/keywords are `_HULL_TRANSFORMATION_` and the existing short forms `_HT?`, `_HT!`, `_HT*` and `_HT^` (more on short from transformations in a second). These expressions signal the use of a transformation to determine the actual value to be rendered. More information about transformations and how to specify and use them can be found in the [HULL docs on transformations](https://github.com/vidispine/hull/blob/main/hull/doc/transformations.md).

### The `_HT?` function

The `_HT?` (or in long form `hull.util.transformation.bool` function) is a first simple transformation example albeit a useful one to determine whether a particular object should be rendered or not at deploy time. For all HULL objects this is controlled via the boolean `enabled` switch, you can set it for each main object defined through HULL. By binding the `enabled` property's value to a `hull.util.transformation.bool` function you can achieve the desired behavior discussed beforehand: use a transformation on `enabled` properties to toggle them to `true` whenever the respective condition has been met (here the `data` section has entries) and `false` otherwise. 

To highlight the usage of transformations, the built-in `hull.util.transformation.bool` transformation used in this context would look like this:

```yaml
enabled: _HULL_TRANSFORMATION_<<<NAME=hull.util.transformation.bool>>><<<CONDITION=
          (index . "$").Values.hull.objects.configmap.addheaders.data>>>
```
where we submit the pointer to the `data` section as a boolean condition (knowing it is false if `data` is empty and true if not) within the CONDITION argument. To really save some typing in the future we are going to use the short form to built-in transformations. They can be viewed as a kind of macro that points to a specifc transformation and omit the argument naming because the transformation only has one argument whose value is supplied directly. The same transformation as above is in short form:

```yaml
enabled: _HT?(index . "$").Values.hull.objects.configmap.addheaders.data
```

which minifies the call to the function and is way handier to type. Again please checkout the [HULL docs on transformations](https://github.com/vidispine/hull/blob/main/hull/doc/transformations.md) for more information.

Now just start again from scratch to add in the transformation that renderes the ConfigMap only on existence of `data`:

```bash
$ echo 'hull:
  objects:
    configmap:
      addheaders:
        enabled: _HT?(index . "$").Values.hull.objects.configmap.addheaders.data
        data: {}
      proxyheaders:
        enabled: _HT?(index . "$").Values.hull.objects.configmap.proxyheaders.data
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

and now there are no ConfigMaps in the output! But this is of course actually correct since no `data` entries were given so far and thus rendering of ConfigMaps should be suppressed.

To add some example `addheaders` and `proxyheaders` to the configuration it is time to bring in another familiar layer of configuration that helps in performing our tests of the functionality and simulate system specific settings in addition to the `values.yaml` defaults. An 'external' or 'system specific' `values.yaml`' is required to vary the configuration on top of the defaults and see the effect on rendering outputs. Before any HULL related processing takes place, the contents of this external file are simply merged entry by entry with the `values.yaml` structure when applied with `-f <file-path>` to any `helm` command. Note that any data that is added via the `-f` switch has precedence over the values.yaml defaults (which is what you'd expect anyways). 

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

and yes: 

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

it worked - you now have dynamically created the `addheaders` ConfigMap just by adding some data to it! Try it the other way around to verify it was not an accident by creating a second 'external configuration' that enables the `proxyheaders` instead and disables the `addheaders`: 

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

where the render output looks like this:

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

Now there is the `proxyheaders` ConfigMap with `data` and no `addheaders` ConfigMap which should suffice to prove that the transformation works! Time to move on to the next ConfigMap pair, the `templates/controller-configmap-tcp.yaml` and `templates/controller-configmap-udp.yaml`:

```bash
$ cat ../ingress-nginx/templates/controller-configmap-tcp.yaml
```

to show the first file's contents:

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

to show the second file's contents:

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

Again check which differences:

```bash
$ diff ../ingress-nginx/templates/controller-configmap-tcp.yaml ../ingress-nginx/templates/controller-configmap-udp.yaml
```

do exist between the two:

```diff
1c1
< {{- if .Values.tcp -}}
---
> {{- if .Values.udp -}}
8,9c8,9
< {{- if .Values.controller.tcp.annotations }}
<   annotations: {{ toYaml .Values.controller.tcp.annotations | nindent 4 }}
---
> {{- if .Values.controller.udp.annotations }}
>   annotations: {{ toYaml .Values.controller.udp.annotations | nindent 4 }}
11c11
<   name: {{ include "ingress-nginx.fullname" . }}-tcp
---
>   name: {{ include "ingress-nginx.fullname" . }}-udp
13c13
< data: {{ tpl (toYaml .Values.tcp) . | nindent 2 }}
---
> data: {{ tpl (toYaml .Values.udp) . | nindent 2 }}
```

Both are also very alike and look similar to the `addheaders` case in structure. Using the same approach of _if there is something in `data:` we render the ConfigMap, otherwise not_ seems reasonable but investigation of further usages shows us that the `tcp` values:

```bash
$ find ../ingress-nginx/templates -type f -print | xargs grep ".Values.tcp"
```

are used quite often throughout the original chart:

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

The same holds true for `udp` values: 

```bash
$ find ../ingress-nginx/templates -type f -print | xargs grep ".Values.udp"
```

which are also referenced in many multiple places in the original chart:
 
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

In cases like these where individual properties are referenced in multiple places (1:n relations) it is often better to define the actual data in a 'single source of truth' fashion same as it is done in the original Helm chart concept with the `.Values.tcp` and `.Values.udp` fields being referencable everywhere. 

HULL can do this too and there is a clearly defined place in the HULL concept for such properties that need to be referenced multiple distinct times for object creation in the `hull.objects` section: the `hull.config.specific` dictionary. This `hull.config.specific` section is specifically intended for this purpose of adding central configuration data that is either singled out for convenient documentation, highlighting and configuration or being used in multiple places within the chart. You can think of this section resembling one of the functions of the `values.yaml` itself within the classic Helm chart approach, namely to centralize access to configuration values. Technically the `hull.config` section is processed before the `hull.objects` section and hence properties defined here and their values (even if derived by a HULL transformations!) are always available for referencing to when defining the objects in the `hull.objects` section.

To cover the dynamic nature of the `tcp` and `udp` entries (it is not known if and how many entries may be provided by the user at deploy/configuration time), the regular key value pair based `data: ` section construction approach does not fit here and it is needed to create the `data:` section with a transformation that reads in the values from the `hull.config.specific` properties and inserts them in the `data` section as key value pairs. Note that it is also possible to mix static keys with dynamically added keys (via transformations) but since there is nothing else in above ConfigMaps `data` section here we can just write `data` completely. This approach has some added complexity but it helps to maintain the 'single source of truth' for crucial values to simplify configuration for the user in the end.

While this modelling reduces the straight-forwardness and transparency of the HULL approach when it comes to configuring object properties, for some configurations (mostly 1:n mappings) it makes more sense to abstract and centralize them for better usability. In other words: with HULL you do not have to literally 'spell out' each projection between `values.yaml` and the rendered objects by putting it into the `template` files but whenever an individual abstraction improves user experience it may be applied.

### Entering the `_HT!` transformation

There are some convenience transformations like the `hull.util.transformation.bool` transformation within HULL but by far the most powerful and flexible is the HULL `hull.util.transformation.tpl` or `_HT!` transformation. It executes the Helm `tpl` function on a given string providing full range of Go templating capabilities as would be the case with Go templating expressions within `templates`. 

Using the `hull.util.transformation.tpl` transformation you can return different datatypes that are expected for the property's values you apply it on:

- string
- int
- bool
- array/list
- dictionary/map

An important aspect to keep in mind when producing arrays and dictionaries with the HULL `hull.util.transformation.tpl` transformation is to use [`flow` style YAML](https://cloudslang-docs.readthedocs.io/en/latest/overview/yaml_overview.html#indentation-scoping) to define the transformation result. This style reduces syntax problems (mostly due to indendation) appearing with the `block` style YAML that is more typically used when dealing with Helm and Kubernetes. Check the next example on how flow style YAML looks like (it is more similar to JSON and does not rely on correct indentation).

To create the `tcp` and `udp` sections now that will centrally hold the data we insert into the existing `values.yaml` the `config.specific` block which holds the `tcp` and `udp` data:

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
  
Time to add the `tpl` transformations to create the ConfigMaps `data:` fields from the now created and referenced sections, the enabled property will be handled by the `hull.util.transformation.bool` transformation as seen previously. The `hull.util.transformation.tpl`'s short form `_HT!` is being used and in the `tpl` argument you construct a flow style dictionary by iterating over the `hull.config.specific.controller.configMaps.tcp.mappings` and `hull.config.specific.controller.configMaps.udp.mappings` respectively. The result of this operation (a dictionary with a dynamic number of entries) is returned as the `data` properties value. An important note to consider: at this point [a HULL `data` structure](https://github.com/vidispine/hull/blob/main/hull/doc/objects_configmaps_secrets.md) needs to be created, not the Kubernetes style `data` structure because HULL expects a HULL object here which is then processed further by HULL in the rendering pipeline. Also note that the transformation usage here comes in form of a dictionary declaration since the data type for the `data` section within HULL is dictionary and we need to stick to this. 

Again use the short form for more compactness when using transformations:

```bash
$ echo '      tcp:
        enabled: _HT?(index . "$").Values.hull.config.specific.controller.configMaps.tcp.mappings
        data: |-
          _HT!{
            {{ range $key,$value :=  (index . "$").Values.hull.config.specific.controller.configMaps.tcp.mappings }}
            {{ $key }}: { inline: {{ $value }} },
            {{ end }}
          }
      udp:
        enabled: _HT?(index . "$").Values.hull.config.specific.controller.configMaps.udp.mappings
        data: |-
          _HT!{
            {{ range $key,$value :=  (index . "$").Values.hull.config.specific.controller.configMaps.udp.mappings }}
            {{ $key }}: { inline: {{ $value }} },
            {{ end }}
          }' >> values.yaml
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
              "67890": some_udp_service' > ../configs/tcp-udp-mappings.yaml
```

Apply on top of the `values.yaml`:

```bash
$ helm template -f ../configs/tcp-udp-mappings.yaml .
```

and you can see the two ConfigMaps being dynamically rendered in case of present data:

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

That's it. You have learned how you can start to leverage the power of transformations - particularly the `hull.util.transformation.tpl` transformation - to improve chart creation and configuration! 

## Specifying ConfigMap and Secret content inside `values.yaml`

You likely noticed that within the 'external configuration' files `data` section the actual values for `data` keys were directly embedded in the `values.yaml` via the `inline` property. While it is possible and in some cases better to import ConfigMap and Secret `data` values from external files via the `files` property, it is handy to be able to specify shorter contents right in place where the ConfigMap is defined. This of course works within `values.yaml` itself and not only in overlays as in the above examples. In the last remaining ConfigMap case you will explore a different way of adding ConfigMap or Secret data by embedding external file contents instead. 

As always, first check the file in question which is the `templates/controller-configmap.yaml`: 

```bash
$ cat ../ingress-nginx/templates/controller-configmap.yaml
```

looking like this:

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

Apparently this file is the central configuration file for the NGinx ingress controller container itself. In it there are four `data` keys that need to be dealt with and since the application likely expects them to exist named exactly like here so their key names cannot be changed in any way:

1. the `add-headers` key and value is set dependent on existence of `controller.addHeaders` entries. The logic seems pretty clear: whenever a `controller.addHeaders` dictionary is defined, an `add-headers` key for it is created in `templates/controller-configmap.yaml` and the value points to the additionally created ConfigMap object defined in `templates/controller-configmap-addheaders.yaml` where the `controller.addHeaders` key-value pairs are stored.

2. the `proxy-set-headers` key and value does the same for `proxyheaders` as what the `add-headers` key does for the `addheaders` ConfigMap.

3. the `allow-snippet-annotations` key whose value comes directly from `controller.allowSnippetAnnotations`. It is a simple boolean value as you can see here:

    ```bash
    $ cat ../ingress-nginx/values.yaml | grep 'allowSnippetAnnotations' -B 10 -A 10
    ```

    which gives:

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
4. the `ssl-dh-param` key which is dependent on `dhParam` being set. The `dhparam` Secret is handled in a minute, for now just treat the 'if data exists create object' condition same as with the ConfigMapsb before.

Any other `data` configuration keys and their values can just be added to the ConfigMap object via HULL's direct configuration possibility in any system-specific `values.yaml`. Hence we don't need to consider the

    ```yaml
    {{- range $key, $value := .Values.controller.config }}
        {{ $key | nindent 2 }}: {{ $value | quote }}
    {{- end }}
    ```

block further. Consumers may just add the key value pairs via the HULL object specification in the system specific YAML like this to add their data:

```yaml
hull:
  objects:
    configmap:
      controller:
        data:
          custom_added_key_1: 
            inline: "Added for demonstration purposes"
          custom_added_key_2: 
            inline: "Also added for demonstration purposes"
```

## Importing ConfigMap and Secret `data` via external files

To explore the options that come with HULL this time you will import the `add-headers` key of the `controller` ConfigMap not via the handy `inline` directive but load it from an external file via the `path` directive. It is really simple, just provide a relative path to a file and it will be rendered as a ConfigMap (or Secret) `data:` entry. For Secrets you don't have to care about Base64 encoding because HULL handles this under its hood, you can just provide the string data as is.
 
As you can see the `add-headers` value is the concatenation of the current namespace with the `addheaders` ConfigMap name. Dynamically computed content of ConfigMap or Secret entries due to usage of template expressions is well suited for supplying it from external files. Especially if you'd have large configuration file like content with many templating expressions it is cleaner to provide the file as external content to not clutter the `values.yaml` unneccesarily.

In this case the content of the `add-headers` key is pretty limited so you could normally also provide it simply `inline`. Using an external file here for the `add-headers` key allows to directly highlight the difference to the `inline` handling of the `proxy-set-headers` key via HULL because functionally the same is happening. As a Bonus you can compare the HULL transformation syntax to the regular templating expression syntax as they are usable within an externally imported file.

Now on to add the `controller` ConfigMap and provide the `add-headers` content via an external file from the `/files` folder and the rest via `inline` instructions as before. First create the `files` folder:

```bash
$ mkdir files
```

and then create the external file content:

```bash
$ echo '{{ .Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $ "COMPONENT" "addheaders") }}' > files/add-headers
```

Now create the full ConfigMap definition for the `controller` as discussed:

```bash
$ echo '      controller:
        data:
          add-headers:
            enabled: _HT?(index . "$").Values.hull.objects.configmap.addheaders.data
            path: files/add-headers
          set-proxy-headers:
            enabled: _HT?(index . "$").Values.hull.objects.configmap.proxyheaders.data
            inline: _HT!{{ (index . "$").Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "proxyheaders") }}
          allow-snippet-annotations:
            inline: "true"
          dhparam.pem:
            enabled: _HT?(index (index . "$").Values.hull.objects.secret.dhparam.data "dhparam.pem").inline
            inline: _HT!{{ (index (index . "$").Values.hull.objects.secret.dhparam.data "dhparam.pem").inline }}' >> values.yaml
```

A comparison of the `inline` transformation of `set-proxy-headers` with the regular Go templating expression in the `path` imported `add-headers` yields the main syntactical differences you need to take care of when using the HULL transformations inside the `values.yaml`:

```yaml
{{ .Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" $ "COMPONENT" "addheaders") }}
```
versus

```yaml
_HT!{{ (index . "$").Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "proxyheaders") }}
```

These are the key differences to notice:

- the HULL `hull.util.transformation.tpl` function is a string that starts with the signaling `_HT!` macro and the content to `tpl`

- Within HULL transformations access to the Helm charts global context is gained via the expression `(index . "$")` instead of just `$` as in regular Helm, this is a technical implication. If you require multiple accesses to the global context within a single transformation you can however simply put something like: 

    ```yaml
    {{ $p := (index . "$") }}
    ```
    at the top of your `tpl` block and thus use `$p` instead of `(index . "$")` within this transformations argument for convenience. 
    
- additionally within the argument all `"` 's may need to be escaped like `\"` when the argument value is wrapped as a string using `"` around the content. If you can omit the string wrapping the escape characters are not required which makes for a simpler writing and reading.

Basically that is all that needs to be considered: you should get similar power of dynamically accessing keys in `values.yaml` and templating logic with the HULL transformation as you would with 'regular' way of using Go templating expressions in the `templates` folder.

## Specifying Secrets with HULL

After this excourse, there is one Secret referenced here at key `ssl-dh-param` which you need to port over before testing our chart again and that is the `templates/dh-param-secret.yaml`:

Running a `cat` on it:

```bash
$ cat ~/ingress-nginx/templates/dh-param-secret.yaml
```

shows the contents:

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

A single `data` key `dhparam.pem` is created along with the Secret itself in case `dhParam` is set in `values.yaml`. Check the `values.yaml` for the definition of `dhParam`:

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

The hit for `dhParam` in the `values.yaml` does not directly indicate the type of the value but the description points to it being a simple string. 

Luckily creating secrets with HULL is exactly the same process as creating ConfigMaps, all required Base64 encoding is done automatically under the hood even when using the `inline` function. It's easy then to add the `dhparam` Secret in the same manner as the ConfigMaps before:

```bash
$ echo '    secret:
      dhparam:
        enabled: _HT?(index (index . "$").Values.hull.objects.secret.dhparam.data "dhparam.pem").inline
        data: 
          dhparam.pem: {}' >> values.yaml
```

At this point feel free to play around with the already created example files in the `config` folder and check if you get the expected output rendered. For example, seeing the `add-headers: default/release-name-ingress-nginx-hull-addheaders` key value pair in the `controller` ConfigMap appear when templating with `-f ../configs/two-add-headers.yaml`. Of course you can also create new ones for testing the functionality. 

## Wrap up

This concludes the subject of handling ConfigMaps and Secrets within HULL. Your `values.yaml` should at this stage look like this:

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
        enabled: _HT?(index . "$").Values.hull.objects.configmap.addheaders.data
        data: {}
      proxyheaders:
        enabled: _HT?(index . "$").Values.hull.objects.configmap.proxyheaders.data
        data: {}
      tcp:
        enabled: _HT?(index . "$").Values.hull.config.specific.controller.configMaps.tcp.mappings
        data: |-
          _HT!{
            {{ range $key,$value :=  (index . "$").Values.hull.config.specific.controller.configMaps.tcp.mappings }}
            {{ $key }}: { inline: {{ $value }} },
            {{ end }}
          }
      udp:
        enabled: _HT?(index . "$").Values.hull.config.specific.controller.configMaps.udp.mappings
        data: |-
          _HT!{
            {{ range $key,$value :=  (index . "$").Values.hull.config.specific.controller.configMaps.udp.mappings }}
            {{ $key }}: { inline: {{ $value }} },
            {{ end }}
          }
      controller:
        data:
          add-headers:
            enabled: _HT?(index . "$").Values.hull.objects.configmap.addheaders.data
            path: files/add-headers
          set-proxy-headers:
            enabled: _HT?(index . "$").Values.hull.objects.configmap.proxyheaders.data
            inline: _HT!{{ (index . "$").Release.Namespace }}/{{ include "hull.metadata.fullname" (dict "PARENT_CONTEXT" (index . "$") "COMPONENT" "proxyheaders") }}
          allow-snippet-annotations:
            inline: "true"
          dhparam.pem:
            enabled: _HT?(index (index . "$").Values.hull.objects.secret.dhparam.data "dhparam.pem").inline
            inline: _HT!{{ (index (index . "$").Values.hull.objects.secret.dhparam.data "dhparam.pem").inline }}
    secret:
      dhparam:
        enabled: _HT?(index (index . "$").Values.hull.objects.secret.dhparam.data "dhparam.pem").inline
        data:
          dhparam.pem: {}
```

Pretty compact for what ground was covered in terms of converted objects and the dynamic functionalities associated with them. The more than a hundred lines of configuration in the original `ingress-nginx` chart that were spent to define the ConfigMaps and the Secret were reduced to a mere fifty and that is even excluding the relevant lines from the original `values.yaml`. That means a reduction of nearly half of the lines of code that require maintenance. Yet at the same time you are gaining the full possibility to overwrite any objects property. When working on the workload objects (Deployments, DaemonSet and Jobs) this reduction of the code to be maintained is even more visible and you will see that you need roughly half the code lines then to achieve even more than what is possible with the original chart.

Now is the right time to store your current `values.yaml` as `values.tutorial-part.yaml` in the charts folder:

```bash
$ cp values.yaml values.tutorial-part.yaml
```

This file is supposed to hold the individual outcome of each tutorial part from now one, concentrating on the aspects that are in focus of the respective tutorial part. 

Additionally a `values.full.yaml` will from now on be maintained and will be built upon in each tutorail part and become the final complete `values.yaml` for the chart. Basically it represents the outcome of the whole tutorial series. Within this constantly growing file all intermediate tutorial results are being added. For this first tutorial where first objects were created it is the same as the `values.intermediate.yaml` obviously but starting from the next tutorial part it will grow further. So at this time just copy the `values.yaml` too:

```bash
$ cp values.yaml values.full.yaml
```

By backing up your progress this way the 'working' `values.yaml` can be reset or modified for each tutorial part to limit the output to look at (since we do not require all objects that were created so far each time) but only those in focus of a tutorial part.

Otherwise that is the end of this tutorial part, the outcome of it is found [here](https://github.com/vidispine/hull-tutorials/tree/test/dev-to/hull/03_configmaps_and_secrets). 

__Thanks for reading and hopefully you found it useful and/or interesting!__