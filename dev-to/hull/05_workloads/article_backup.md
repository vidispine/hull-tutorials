Adding Workload objects
In this Part of our tutorial we will investigate the workload objects. A workload object has a pod at its core in which containers run code. Most important workload types are Deployments, DaemonSets, StatefulSets, Jobs and CronJobs so let us first check which of those we can find:
find ../ingress-nginx/templates -type f -iregex '.*\(deployment\|daemonset\|statefulset\|job\).*' | sort
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
Two jobs, two deployments and one DaemonSet to look at … starting with the templates/admission-webhooks/job-patch/job-createSecret.yaml Job:
cat ../ingress-nginx/templates/admission-webhooks/job-patch/job-createSecret.yaml
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
Here one of the major strengths of HULL comes into play: setting any available object property can be done directly without the need to write any repetitive boilerplate code which goes like 'if this is defined in values.yaml do insert it here' and does nothing else - most Helm charts are full of these 1:1 mappings between values.yaml properties and object properties in the /templates.

Luckily all of the following blocks below can hence be ignored for our conversion because we can configure the properties without any additional action at configuration/deployment time if needed:
{{- if .Values.controller.admissionWebhooks.patch.podAnnotations }}
      annotations: {{ toYaml .Values.controller.admissionWebhooks.patch.podAnnotations | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.admissionWebhooks.patch.priorityClassName }}
      priorityClassName: {{ .Values.controller.admissionWebhooks.patch.priorityClassName }}
    {{- end }}
    {{- if .Values.imagePullSecrets }}
      imagePullSecrets: {{ toYaml .Values.imagePullSecrets | nindent 8 }}
    {{- end }}
          {{- if .Values.controller.admissionWebhooks.createSecretJob.resources }}
          resources: {{ toYaml .Values.controller.admissionWebhooks.createSecretJob.resources | nindent 12 }}
          {{- end }}
    {{- if .Values.controller.admissionWebhooks.patch.nodeSelector }}
      nodeSelector: {{ toYaml .Values.controller.admissionWebhooks.patch.nodeSelector | nindent 8 }}
    {{- end }}
    {{- if .Values.controller.admissionWebhooks.patch.tolerations }}
      tolerations: {{ toYaml .Values.controller.admissionWebhooks.patch.tolerations | nindent 8 }}
    {{- end }}
A word on imagePullSecrets though: it is very easy to configure and use them with HULL. By default, all registry objects that are created in HULL will be added as imagePullSecrets to the pod specs so images are always pullable even if they are stored on a secured registry. If this behavior is undesired, you can disable this feature simply by setting createImagePullSecretsFromRegistries to false in the HULL configuration like this:
hull:
  config:
    general:
      createImagePullSecretsFromRegistries: false
You can however combine this feature with the possibility to set all pods registry fields to a common value. This is allowed for using a fixed registry server name (for which the docker registry secret is deployed outside of the Helm chart) or for using the server defined in the first registry secret that is configured in the same chart.

To provide an example of a real life use case imagine you host all Docker images referenced in your Helm chart in one place, meaning in one Docker registry. Now you may want to switch Docker registry providers and hence address or copy over all Docker images to an airgapped system and host them in a local secured registry such as Harbor. In both cases you may want to centrally change the registry servers address without having to configure each pod specs imagePullSecrets individually for which these options are exactly for.

Refer to the documentation and the following config switches:
hull:
  config:
    general:
      defaultImageRegistryServer: 
      defaultImageRegistryToFirstRegistrySecretServer:
The serviceAccountName configuration was discussed in detail in the tutorial part that dealt with RBAC configuration options so it will not be highlighted again here.

Now to the other aspects of this job:

the whole object creation is being controlled by having webhooks generally enabled and the patch webhook in particular ({{ if and .Values.controller.admissionWebhooks.enabled .Values.controller.admissionWebhoooks.patch.enabled }} condition). Those conditions govern creation of all objects in the /templates/admission-webhooks/job-patch folder so it makes sense to control the condition in a central place in HULL as well. As already discussed, the hull.config.specific section is the indended place for defining globally relevant properties for your chart. Hence it is feasible to add boolean properties admissionWebhooks.enabled and admissionWebhooks.patch.enabled to hull.objects.specific so that the object rendering of the webhooks is also controlled via the enabled properties same as in the original chart.

Helm hooks such as:
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-delete-policy": before-hook-creation,hook- succeeded
are simply added to the Job via annotations.
a Capabilities check exists to conditionally include ttlSecondsAfterFinished depending on the Kubernetes cluster API version. With HULL being tied to a specific minimum Kubernetes version checks, the property is either available or not:
  {{- if .Capabilities.APIVersions.Has "batch/v1alpha1" }}
    # Alpha feature since k8s 1.12
    ttlSecondsAfterFinished: 0
  {{- end }}
the name suffix will be changed from admission-create to admissioncreate for simplicity
the args section requires using a HULL transformation to create an array of arguments that match the dynamic nature of the argument construction.
Putting everything together the job-createSecret in the HULL chart may look like this: