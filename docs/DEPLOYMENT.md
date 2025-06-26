# Django Ledger Deployment Guide with ArgoCD

This guide provides step-by-step instructions for deploying the Django Ledger application using Helm and ArgoCD on a Minikube cluster.

## Prerequisites

- Minikube installed
- kubectl configured
- Helm installed
- ArgoCD CLI installed

## Creating a Helm Chart from Scratch

### 1. Initialize a New Helm Chart

```bash
# Create a new directory for your Helm chart
mkdir django-ledger
cd django-ledger

# Initialize a new Helm chart
helm create django-ledger

# The chart structure will be created as follows:
# django-ledger/
# ├── Chart.yaml
# ├── values.yaml
# ├── templates/
# │   ├── deployment.yaml
# │   ├── service.yaml
# │   ├── ingress.yaml
# │   ├── hpa.yaml
# │   ├── _helpers.tpl
# │   ├── NOTES.txt
# │   └── serviceaccount.yaml
# └── charts/
```

### 2. Update Chart.yaml

```bash
# Edit Chart.yaml with your application details
cat > Chart.yaml << EOF
apiVersion: v2
name: django-ledger
description: A Helm chart for Django Ledger application
type: application
version: 0.1.0
appVersion: "1.0.0"
EOF
```

### 3. Create Required Templates

```bash
# Create deployment template
cat > templates/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "django-ledger.fullname" . }}
  labels:
    {{- include "django-ledger.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "django-ledger.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "django-ledger.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 8000
          env:
            {{- range $key, $value := .Values.django.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          volumeMounts:
            - name: static-files
              mountPath: /data/static
            - name: media-files
              mountPath: /data/media
      volumes:
        - name: static-files
          persistentVolumeClaim:
            claimName: {{ include "django-ledger.fullname" . }}-static
        - name: media-files
          persistentVolumeClaim:
            claimName: {{ include "django-ledger.fullname" . }}-media
EOF

# Create service template
cat > templates/service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: {{ include "django-ledger.fullname" . }}
  labels:
    {{- include "django-ledger.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8000
      protocol: TCP
  selector:
    {{- include "django-ledger.selectorLabels" . | nindent 4 }}
EOF

# Create PostgreSQL deployment
cat > templates/postgres.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "django-ledger.fullname" . }}-postgres
  labels:
    {{- include "django-ledger.labels" . | nindent 4 }}
    app.kubernetes.io/component: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      {{- include "django-ledger.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: postgres
  template:
    metadata:
      labels:
        {{- include "django-ledger.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: postgres
    spec:
      containers:
        - name: postgres
          image: "{{ .Values.postgres.image.repository }}:{{ .Values.postgres.image.tag }}"
          imagePullPolicy: {{ .Values.postgres.image.pullPolicy }}
          ports:
            - containerPort: 5432
          env:
            {{- range $key, $value := .Values.postgres.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: {{ include "django-ledger.fullname" . }}-postgres
EOF

# Create PostgreSQL service
cat > templates/postgres-service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: {{ include "django-ledger.fullname" . }}-postgres
  labels:
    {{- include "django-ledger.labels" . | nindent 4 }}
    app.kubernetes.io/component: postgres
spec:
  ports:
    - port: {{ .Values.postgres.service.port }}
      targetPort: 5432
      protocol: TCP
  selector:
    {{- include "django-ledger.selectorLabels" . | nindent 4 }}
    app.kubernetes.io/component: postgres
EOF

# Create PVC templates
cat > templates/pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "django-ledger.fullname" . }}-static
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "django-ledger.fullname" . }}-media
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "django-ledger.fullname" . }}-postgres
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.postgres.persistence.size }}
EOF
```

### 4. Create Helper Templates

```bash
# Create _helpers.tpl
cat > templates/_helpers.tpl << EOF
{{/*
Expand the name of the chart.
*/}}
{{- define "django-ledger.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "django-ledger.fullname" -}}
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
{{- define "django-ledger.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "django-ledger.labels" -}}
helm.sh/chart: {{ include "django-ledger.chart" . }}
{{ include "django-ledger.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "django-ledger.selectorLabels" -}}
app.kubernetes.io/name: {{ include "django-ledger.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
EOF
```

### 5. Create values.yaml

```bash
# Create values.yaml with default values
cat > values.yaml << EOF
replicaCount: 1

image:
  repository: runtesting/django-ledger
  tag: "1.0"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 8000

django:
  env:
    DEBUG: "False"
    SECRET_KEY: "jollof12345"
    ALLOWED_HOSTS: "localhost,127.0.0.1,minikube.local"
    DATABASE_URL: "postgres://postgres:postgres@django-ledger-postgres:5432/django_ledger"
    STATIC_ROOT: "/data/static"
    STATIC_URL: "/static/"
    MEDIA_ROOT: "/data/media"
    MEDIA_URL: "/media/"
    CSRF_TRUSTED_ORIGINS: "http://localhost,http://127.0.0.1,http://minikube.local"
    SECURE_SSL_REDIRECT: "False"
    SESSION_COOKIE_SECURE: "False"
    CSRF_COOKIE_SECURE: "False"
    DJANGO_SETTINGS_MODULE: "django_ledger_starter.settings"

postgres:
  image:
    repository: postgres
    tag: "13"
    pullPolicy: IfNotPresent
  service:
    port: 5432
  persistence:
    size: 1Gi
  env:
    POSTGRES_DB: django_ledger
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
EOF
```

### 6. Test the Chart

```bash
# Lint the chart
helm lint .

# Do a dry run
helm install django-ledger . --dry-run --debug

# Install the chart
helm install django-ledger .
```

## Deployment Steps

### 1. Start Minikube and Install ArgoCD

```bash
# Start Minikube
minikube start

# Create ArgoCD namespace and install it
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD pods to be ready
kubectl get pods -n argocd -w
```

### 2. Package and Serve Helm Chart

```bash
# Navigate to your Helm chart directory
cd django-ledger

# Package the Helm chart
helm package .

# Create repository index
helm repo index . --url https://github.com/fred4impact/django-ledger-repo/raw/main/django-ledger

# Add the repository to Helm
helm repo add django-ledger https://github.com/fred4impact/django-ledger-repo/raw/main/django-ledger

# Update Helm repositories
helm repo update
```

### 3. Set Up ArgoCD Access

```bash
# Port-forward ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0

# Get ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 4. Add Helm Repository to ArgoCD

```bash
# Add the local Helm repository
argocd repo add https://github.com/fred4impact/django-ledger-repo/raw/main/django-ledger --type helm --name django-ledger
```

### 5. Deploy the Application

```bash
# Apply the ArgoCD application manifest
kubectl apply -f django-ledger-argocd.yaml
```

### 6. Verify Deployment

```bash
# Check ArgoCD application status
argocd app get django-ledger

# Check pods
kubectl get pods

# Check services
kubectl get services

# Get the service URL
minikube service django-ledger-django --url
```

### 7. Access the Application

```bash
# Option 1: Using port-forward
kubectl port-forward svc/django-ledger-django 8000:8000

# Option 2: Using Minikube service URL
minikube service django-ledger-django --url
```

### 8. Monitor Logs

```bash
# View application logs
kubectl logs -l app.kubernetes.io/name=django-ledger

# View ArgoCD logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server
```

### 9. Access ArgoCD UI

- Open browser and go to: https://localhost:8080
- Login with:
  - Username: admin
  - Password: (the password you got from step 3)

### 10. Troubleshooting Commands

```bash
# Check pod status
kubectl get pods -o wide

# Check service endpoints
kubectl get endpoints

# Check events
kubectl get events --sort-by='.lastTimestamp'

# Check ArgoCD sync status
argocd app sync django-ledger

# Check database connection
kubectl exec -it $(kubectl get pod -l app.kubernetes.io/name=django-ledger -o jsonpath="{.items[0].metadata.name}") -- python manage.py check
```

### 11. Clean Up

```bash
# Delete the ArgoCD application
kubectl delete -f django-ledger-argocd.yaml

# Delete the Helm release
helm uninstall django-ledger
```

## Configuration

### Helm Chart Values

The main configuration values are in `django-ledger/values.yaml`:

```yaml
image:
  repository: runtesting/django-ledger
  tag: "1.0"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 8000

django:
  env:
    DEBUG: "False"
    SECRET_KEY: "jollof12345"
    ALLOWED_HOSTS: "localhost,127.0.0.1,minikube.local"
    DATABASE_URL: "postgres://postgres:postgres@django-ledger-postgres:5432/django_ledger"
    STATIC_ROOT: "/data/static"
    STATIC_URL: "/static/"
    MEDIA_ROOT: "/data/media"
    MEDIA_URL: "/media/"
    CSRF_TRUSTED_ORIGINS: "http://localhost,http://127.0.0.1,http://minikube.local"
    SECURE_SSL_REDIRECT: "False"
    SESSION_COOKIE_SECURE: "False"
    CSRF_COOKIE_SECURE: "False"
    DJANGO_SETTINGS_MODULE: "django_ledger_starter.settings"

postgres:
  image:
    repository: postgres
    tag: "13"
    pullPolicy: IfNotPresent
  service:
    port: 5432
  persistence:
    size: 1Gi
  env:
    POSTGRES_DB: django_ledger
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
```

### ArgoCD Application Manifest

The ArgoCD application configuration is in `django-ledger-argocd.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: django-ledger
  namespace: argocd
spec:
  project: default
  source:
    chart: django-ledger
    repoURL: https://github.com/fred4impact/django-ledger-repo/raw/main/django-ledger
    targetRevision: 0.1.0
    helm:
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

## Verification Checklist

- [ ] ArgoCD UI shows application as "Healthy"
- [ ] All pods are running
- [ ] Services are properly configured
- [ ] PersistentVolumeClaims are bound
- [ ] Application is accessible via service URL
- [ ] Database connection is working
- [ ] Static files are served correctly

## Troubleshooting

### Common Issues

1. Database Connection Issues
   - Verify PostgreSQL pod is running: `kubectl get pods | grep postgres`
   - Check database logs: `kubectl logs -l app.kubernetes.io/name=django-ledger-postgres`
   - Ensure DATABASE_URL in values.yaml points to correct service name

2. Application Access Issues
   - Verify service is running: `kubectl get services`
   - Check application logs: `kubectl logs -l app.kubernetes.io/name=django-ledger`
   - Ensure port-forwarding is working: `kubectl port-forward svc/django-ledger-django 8000:8000`

3. ArgoCD Sync Issues
   - Check ArgoCD application status: `argocd app get django-ledger`
   - Verify repository access: `argocd repo list`
   - Check ArgoCD logs: `kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server`

## Security Notes

1. The `SECRET_KEY`