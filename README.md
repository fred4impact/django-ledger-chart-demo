# Helm & ArgoCD Deployment Guide for Django + PostgreSQL

This guide walks you through deploying your Django app (with PostgreSQL) on Kubernetes using a GitOps workflow with Helm and ArgoCD.

---

## 1. Prerequisites
- Kubernetes cluster (e.g., Minikube, GKE, EKS, etc.)
- `kubectl` configured
- [Helm](https://helm.sh/) installed (used for templating and packaging)
- [ArgoCD](https://argo-cd.readthedocs.io/) installed (see below)
- A Git repository to store your configuration files.

---

## 2. Create and Package Your Django Helm Chart

First, create a Helm chart for your Django application.
```sh
helm create django-ledger-chart
```
- This creates a new chart in the `django-ledger-chart/` directory.
- Update the following files as needed:
  - `values.yaml`: Set your image, environment variables, etc.
  - `templates/deployment.yaml`: Define your app's deployment.
  - `templates/service.yaml`: Expose your Django app (usually port 8000).

Once your chart is configured, package it. This step is optional if ArgoCD has direct access to your chart source files in Git, but it's good practice for versioning.
```sh
helm package ../django-ledger-chart
```
- This creates a `.tgz` package for your chart.

---

## 3. Install ArgoCD (if not already installed)
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Expose ArgoCD API server (choose one):
- **Port-forward:**
  ```sh
  kubectl port-forward svc/argocd-server -n argocd 8080:443
  ```
- **Ingress/LoadBalancer:** (see ArgoCD docs)

---

## 4. Deploy with ArgoCD

Instead of using `helm install`, we will define our applications declaratively in YAML files. This is the core of the GitOps approach. You should commit these files to your Git repository.

### a. Create the PostgreSQL Application

First, add the Bitnami Helm repository locally so Helm can fetch chart info. ArgoCD will use the repository URL directly.
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

Create a file named `postgres-application.yaml`. This tells ArgoCD how to deploy PostgreSQL.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-postgres
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://charts.bitnami.com/bitnami
    chart: postgresql
    targetRevision: 15.x.x # Pin to a specific chart version for stability
    helm:
      values: |
        auth:
          postgresPassword: "postgres"
          username: "postgres"
          password: "postgres"
          database: "django_ledger"
  destination:
    server: https://kubernetes.default.svc
    namespace: default # Deploy PostgreSQL to the 'default' namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### b. Create the Django Application

Create a file named `django-application.yaml`. This tells ArgoCD where your Django app's configuration is located in your Git repo.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: django-ledger-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: <your-git-repo-url> # CHANGE THIS to your Git repository URL
    targetRevision: HEAD
    path: django-ledger-chart # Path within your repo to the Django chart
  destination:
    server: https://kubernetes.default.svc
    namespace: default # Deploy the app to the 'default' namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
- **Important:** In your Django chart's `values.yaml`, ensure the database host is configured to connect to `my-postgres`, which is the service name of our PostgreSQL deployment.

### c. Apply the Applications

Apply both manifests to your cluster. ArgoCD will detect them and start the deployment process.

```sh
kubectl apply -f postgres-application.yaml
kubectl apply -f django-application.yaml
```
- You can now view and manage both applications from the ArgoCD dashboard.

---

## 5. Test & Verify
- Check pods:
  ```sh
  kubectl get pods -n default
  ```
- Get service info:
  ```sh
  kubectl get svc -n default
  ```
- Port-forward to access Django app:
  ```sh
  kubectl port-forward svc/django-ledger-chart 8000:8000 -n default
  ```
- Visit: [http://localhost:8000](http://localhost:8000)

---

## Notes
- With this approach, you do not need to manually run `helm install` or `helm upgrade`.
- All configuration is stored in Git. To make changes, you update the files in your repository, commit, and push. ArgoCD will automatically sync the changes to your cluster.
- For production, ensure you manage secrets securely (e.g., with a tool like Vault or Sealed Secrets) instead of putting plain text passwords in the YAML files.

---

## References
- [Helm Docs](https://helm.sh/docs/)
- [Bitnami PostgreSQL Chart](https://artifacthub.io/packages/helm/bitnami/postgresql)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)

---

## DevOps Project: GitOps Deployment of Django + PostgreSQL on Kubernetes with Helm & ArgoCD

- Designed and implemented a production-ready deployment pipeline for a Django web application with a PostgreSQL backend on Kubernetes.
- Authored and maintained Helm charts to package and template Kubernetes manifests for the Django application, ensuring modular and reusable infrastructure-as-code.
- Leveraged ArgoCD for GitOps-based continuous deployment, enabling automated, declarative application delivery and version control of all Kubernetes resources.
- Automated the provisioning of supporting resources (PersistentVolumes, PersistentVolumeClaims, ConfigMaps, Secrets) and database migration jobs to ensure reliable, repeatable environment setup.
- Wrote comprehensive documentation and troubleshooting guides to streamline onboarding and maintenance for team members.
- Implemented best practices for secret management, resource namespacing, and application monitoring within Kubernetes.
- Utilized kubectl and Helm CLI for cluster management, resource debugging, and application lifecycle operations.
- Ensured high availability and scalability by configuring Kubernetes services, deployments, and ingress resources.

**Key Technologies:** Kubernetes, Helm, ArgoCD, Django, PostgreSQL, GitOps, Docker, YAML, kubectl

---

## Key Project Summary

Deployed a Django web application with a PostgreSQL database on Kubernetes using Helm for templating and ArgoCD for GitOps-based continuous delivery. Automated the setup of all supporting resources, implemented best practices for security and scalability, and documented the entire workflow for streamlined DevOps operations.
