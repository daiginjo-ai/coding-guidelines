# Helm Charts Guidelines

This document outlines the standards and best practices for Kubernetes deployment using Helm charts in the daiginjo-ai organization.

## Chart Structure

```
helm/
├── charts/                 # Chart dependencies (if any)
├── templates/              # Kubernetes manifest templates
│   ├── deployment.yaml     # Application deployment
│   ├── service.yaml        # Service definition
│   ├── ingress.yaml        # Ingress configuration
│   ├── configmap.yaml      # Configuration data
│   ├── secret.yaml         # Sensitive data
│   ├── serviceaccount.yaml # Service account
│   ├── hpa.yaml           # Horizontal Pod Autoscaler
│   ├── pdb.yaml           # Pod Disruption Budget
│   └── NOTES.txt          # Post-install notes
├── values.yaml             # Default values
├── values-dev.yaml         # Development environment
├── values-staging.yaml     # Staging environment
├── values-prod.yaml        # Production environment
├── Chart.yaml              # Chart metadata
└── README.md              # Chart documentation
```

## Chart.yaml Configuration

```yaml
apiVersion: v2
name: my-application
description: A Helm chart for my-application
type: application
version: 0.1.0
appVersion: "1.0.0"
home: https://github.com/daiginjo-ai/my-application
sources:
  - https://github.com/daiginjo-ai/my-application
maintainers:
  - name: Daiginjo AI Team
    email: devops@daiginjo.ai
keywords:
  - web
  - api
  - microservice
```

## Values Structure

### Default values.yaml
```yaml
# Default configuration
replicaCount: 1

image:
  repository: my-application
  pullPolicy: IfNotPresent
  tag: ""

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}
podLabels: {}

podSecurityContext:
  fsGroup: 2000

securityContext:
  capabilities:
    drop:
    - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000

service:
  type: ClusterIP
  port: 80
  targetPort: 3000

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

# Application-specific configuration
config:
  nodeEnv: production
  logLevel: info
  database:
    host: ""
    port: 5432
    name: ""
    ssl: true

secrets:
  database:
    username: ""
    password: ""
  jwt:
    secret: ""

# Health checks
healthcheck:
  enabled: true
  liveness:
    path: /health
    port: 3000
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    path: /ready
    port: 3000
    initialDelaySeconds: 5
    periodSeconds: 5
```

### Environment-specific values

#### values-dev.yaml
```yaml
replicaCount: 1

image:
  tag: "dev"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-staging"
  hosts:
    - host: my-app-dev.daiginjo.ai
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: my-app-dev-tls
      hosts:
        - my-app-dev.daiginjo.ai

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

config:
  nodeEnv: development
  logLevel: debug
  database:
    host: postgres-dev.daiginjo.ai
    name: myapp_dev
```

#### values-prod.yaml
```yaml
replicaCount: 3

image:
  tag: "v1.0.0"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/rate-limit: "100"
  hosts:
    - host: api.daiginjo.ai
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-daiginjo-ai-tls
      hosts:
        - api.daiginjo.ai

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

config:
  nodeEnv: production
  logLevel: warn
  database:
    host: postgres-prod.daiginjo.ai
    name: myapp_prod

podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

## Template Best Practices

### Deployment Template
```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-application.fullname" . }}
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-application.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "my-application.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-application.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- if .Values.healthcheck.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthcheck.liveness.path }}
              port: {{ .Values.healthcheck.liveness.port }}
            initialDelaySeconds: {{ .Values.healthcheck.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthcheck.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthcheck.readiness.path }}
              port: {{ .Values.healthcheck.readiness.port }}
            initialDelaySeconds: {{ .Values.healthcheck.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthcheck.readiness.periodSeconds }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
            - name: NODE_ENV
              value: {{ .Values.config.nodeEnv | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            - name: DATABASE_HOST
              value: {{ .Values.config.database.host | quote }}
            - name: DATABASE_PORT
              value: {{ .Values.config.database.port | quote }}
            - name: DATABASE_NAME
              value: {{ .Values.config.database.name | quote }}
            - name: DATABASE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-application.fullname" . }}-secret
                  key: database-username
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "my-application.fullname" . }}-secret
                  key: database-password
          envFrom:
            - configMapRef:
                name: {{ include "my-application.fullname" . }}-config
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

### Service Template
```yaml
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-application.fullname" . }}
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "my-application.selectorLabels" . | nindent 4 }}
```

### ConfigMap Template
```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-application.fullname" . }}-config
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
data:
  PORT: {{ .Values.service.targetPort | quote }}
  DATABASE_SSL: {{ .Values.config.database.ssl | quote }}
  {{- range $key, $value := .Values.config }}
  {{- if not (kindIs "map" $value) }}
  {{ $key | upper }}: {{ $value | quote }}
  {{- end }}
  {{- end }}
```

### Secret Template
```yaml
# templates/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "my-application.fullname" . }}-secret
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
type: Opaque
data:
  database-username: {{ .Values.secrets.database.username | b64enc }}
  database-password: {{ .Values.secrets.database.password | b64enc }}
  jwt-secret: {{ .Values.secrets.jwt.secret | b64enc }}
```

### Ingress Template
```yaml
# templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-application.fullname" . }}
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if and .Values.ingress.className (semverCompare ">=1.18-0" .Capabilities.KubeVersion.GitVersion) }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            {{- if and .pathType (semverCompare ">=1.18-0" $.Capabilities.KubeVersion.GitVersion) }}
            pathType: {{ .pathType }}
            {{- end }}
            backend:
              {{- if semverCompare ">=1.19-0" $.Capabilities.KubeVersion.GitVersion }}
              service:
                name: {{ include "my-application.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
              {{- else }}
              serviceName: {{ include "my-application.fullname" $ }}
              servicePort: {{ $.Values.service.port }}
              {{- end }}
          {{- end }}
    {{- end }}
{{- end }}
```

## Helper Templates

### _helpers.tpl
```yaml
{{/*
Expand the name of the chart.
*/}}
{{- define "my-application.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "my-application.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "my-application.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "my-application.labels" -}}
helm.sh/chart: {{ include "my-application.chart" . }}
{{ include "my-application.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "my-application.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-application.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Create the name of the service account to use
*/}}
{{- define "my-application.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-application.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

## Security Best Practices

### Pod Security Standards
```yaml
# Apply Pod Security Standards
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 2000

securityContext:
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
```

### Network Policies
```yaml
# templates/networkpolicy.yaml
{{- if .Values.networkPolicy.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: {{ include "my-application.fullname" . }}
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
spec:
  podSelector:
    matchLabels:
      {{- include "my-application.selectorLabels" . | nindent 6 }}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: {{ .Values.networkPolicy.allowedNamespaces }}
      ports:
        - protocol: TCP
          port: {{ .Values.service.targetPort }}
  egress:
    - to: []
      ports:
        - protocol: TCP
          port: 5432  # Database
        - protocol: TCP
          port: 6379  # Redis
        - protocol: TCP
          port: 443   # HTTPS
        - protocol: TCP
          port: 53    # DNS
        - protocol: UDP
          port: 53    # DNS
{{- end }}
```

## Monitoring and Observability

### ServiceMonitor for Prometheus
```yaml
# templates/servicemonitor.yaml
{{- if .Values.monitoring.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "my-application.fullname" . }}
  labels:
    {{- include "my-application.labels" . | nindent 4 }}
spec:
  selector:
    matchLabels:
      {{- include "my-application.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
{{- end }}
```

## Deployment Guidelines

### Version Management
- Use semantic versioning for chart versions
- Tag images with specific versions, not `latest`
- Use `appVersion` in Chart.yaml to track application version

### Testing
```bash
# Lint the chart
helm lint ./helm

# Template and validate
helm template my-app ./helm --values ./helm/values-dev.yaml

# Dry run
helm install my-app ./helm --values ./helm/values-dev.yaml --dry-run

# Test installation
helm test my-app
```

### Deployment Commands
```bash
# Development
helm upgrade --install my-app ./helm \
  --values ./helm/values-dev.yaml \
  --namespace development \
  --create-namespace

# Production
helm upgrade --install my-app ./helm \
  --values ./helm/values-prod.yaml \
  --namespace production \
  --create-namespace
```

## Documentation Requirements

### Chart README.md
- Installation instructions
- Configuration options
- Examples for different environments
- Troubleshooting guide

### NOTES.txt
```yaml
1. Get the application URL by running these commands:
{{- if .Values.ingress.enabled }}
{{- range $host := .Values.ingress.hosts }}
  {{- range .paths }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ $host.host }}{{ .path }}
  {{- end }}
{{- end }}
{{- else if contains "NodePort" .Values.service.type }}
  export NODE_PORT=$(kubectl get --namespace {{ .Release.Namespace }} -o jsonpath="{.spec.ports[0].nodePort}" services {{ include "my-application.fullname" . }})
  export NODE_IP=$(kubectl get nodes --namespace {{ .Release.Namespace }} -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT
{{- else if contains "LoadBalancer" .Values.service.type }}
     NOTE: It may take a few minutes for the LoadBalancer IP to be available.
           You can watch the status of by running 'kubectl get --namespace {{ .Release.Namespace }} svc -w {{ include "my-application.fullname" . }}'
  export SERVICE_IP=$(kubectl get svc --namespace {{ .Release.Namespace }} {{ include "my-application.fullname" . }} --template "{{"{{ range (index .status.loadBalancer.ingress 0) }}{{.}}{{ end }}"}}")
  echo http://$SERVICE_IP:{{ .Values.service.port }}
{{- else if contains "ClusterIP" .Values.service.type }}
  export POD_NAME=$(kubectl get pods --namespace {{ .Release.Namespace }} -l "app.kubernetes.io/name={{ include "my-application.name" . }},app.kubernetes.io/instance={{ .Release.Name }}" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace {{ .Release.Namespace }} $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace {{ .Release.Namespace }} port-forward $POD_NAME 8080:$CONTAINER_PORT
{{- end }}
```