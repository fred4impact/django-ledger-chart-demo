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

### What are Chart.yaml, values.yaml, and the templates?

- **Chart.yaml**: Basic info about your Helm chart (name, version, description).
- **values.yaml**: Default settings for your app (image, env vars, ports, etc). You can override these when you install the chart.
- **templates/deployment.yaml**: Defines how your app runs on Kubernetes (number of pods, container image, env vars, etc).
- **templates/service.yaml**: Exposes your app inside the cluster (creates a Service so other apps or users can reach it).
- **templates/ingress.yaml**: (Optional) Lets you expose your app to the internet (creates an Ingress for external access).

---

## 3. Install ArgoCD (if not already installed)
```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

t1wk-kS341nAbKUX
```

Expose ArgoCD API server (choose one):
- **Port-forward:**

```sh
  kubectl port-forward svc/argocd-server -n argocd 8080:443
```
- **Ingress/LoadBalancer:** (see ArgoCD docs)

---

## 4. **IMPORTANT: Deploy Supporting Resources FIRST**

**Before deploying your ArgoCD applications, you MUST create all supporting resources (PVs, PVCs, ConfigMaps, Secrets, and migration jobs) in the same namespace as your application.**

### Step 4a: Apply PersistentVolumes (PVs)
```sh
kubectl apply -f manifests/pv-postgres.yml -n test
kubectl apply -f manifests/pv-static.yml -n test
```

### Step 4b: Apply PersistentVolumeClaims (PVCs)
```sh
kubectl apply -f manifests/pvc-postgres.yml -n test
kubectl apply -f manifests/pvc-static.yml -n test
```

### Step 4c: Apply ConfigMap and Secret
```sh
kubectl apply -f manifests/configmap.yml -n test
kubectl apply -f manifests/secret.yml -n test
```

### Step 4d: Verify Resources Are Ready
```sh
# Check if PVCs are bound
kubectl get pvc -n test

# Check if ConfigMap and Secret exist
kubectl get configmap -n test
kubectl get secret -n test
```

**Note:** If PVCs are stuck in "Pending" status, ensure:
- The corresponding PVs exist and are available
- The storage class is correct for your cluster
- The access modes match between PV and PVC

---

## 5. Deploy with ArgoCD

Now that all supporting resources are in place, deploy your applications using ArgoCD.

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
    targetRevision: 16.x.x # Pin to a specific chart version for stability
    helm:
      values: |
        auth:
          postgresPassword: "postgres"
          username: "postgres"
          password: "postgres"
          database: "django_ledger"
  destination:
    server: https://kubernetes.default.svc
    namespace: test # Deploy PostgreSQL to the 'test' namespace
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
    namespace: test # Deploy the app to the 'test' namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
- **Important:** In your Django chart's `values.yaml`, ensure the database host is configured to connect to `my-postgres-postgresql`, which is the service name of our PostgreSQL deployment.

### c. Apply the Applications

Apply both manifests to your cluster. ArgoCD will detect them and start the deployment process.

```sh
kubectl apply -f postgres-application.yaml -n argocd
kubectl apply -f django-application.yaml -n argocd
```

- You can now view and manage both applications from the ArgoCD dashboard.

---

## 6. Run Database Migration Job

After both applications are deployed and running, run the migration job to set up the database and collect static files:

```sh
kubectl apply -f manifests/django-migration-job.yml -n test
```

**Note:** The migration job will:
- Wait for the PostgreSQL database to be ready
- Run Django migrations
- Collect static files to the static PVC

---

## 7. Test & Verify
- Check pods:
  ```sh
  kubectl get pods -n test
  ```
- Get service info:
  ```sh
  kubectl get svc -n test
  ```
- Port-forward to access Django app:
  ```sh
  kubectl port-forward svc/django-ledger-chart 8000:8000 -n test
  ```
- Visit: [http://localhost:8000](http://localhost:8000)

---

## 8. Troubleshooting

### Common Issues and Solutions

#### 1. Pod Scheduling Issues
If you see `FailedScheduling` errors about missing PVCs:
```sh
# Check if PVCs exist in the correct namespace
kubectl get pvc -n test

# Apply missing PVCs
kubectl apply -f manifests/pvc-static.yml -n test
kubectl apply -f manifests/pvc-postgres.yml -n test
```

#### 2. Container Configuration Errors
If you see `CreateContainerConfigError` about missing ConfigMap or Secret:
```sh
# Check if resources exist
kubectl get configmap -n test
kubectl get secret -n test

# Apply missing resources
kubectl apply -f manifests/configmap.yml -n test
kubectl apply -f manifests/secret.yml -n test

# Restart the pod to pick up new resources
kubectl delete pod <pod-name> -n test
```

#### 3. Static Files Not Loading
If static files are not being served:
```sh
# Check if static PVC is bound
kubectl get pvc django-static-pvc -n test

# Check if migration job ran successfully
kubectl get jobs -n test
kubectl logs job/django-migrate -n test

# Re-run migration job if needed
kubectl delete job django-migrate -n test
kubectl apply -f manifests/django-migration-job.yml -n test
```

#### 4. PostgreSQL Version Conflicts
If PostgreSQL is in `CrashLoopBackOff` with version compatibility errors:
```sh
# Check PostgreSQL logs
kubectl logs my-postgres-postgresql-0 -n test

# If you see "database files are incompatible with server" error:
# Delete the PostgreSQL PVC to start fresh
kubectl delete pvc data-my-postgres-postgresql-0 -n test
kubectl delete pod my-postgres-postgresql-0 -n test
```

#### 5. Database Connection Issues
If Django can't connect to PostgreSQL:
```sh
# Check if PostgreSQL service exists
kubectl get svc -n test | grep postgres

# Check if PostgreSQL pod is running
kubectl get pods -n test | grep postgres

# Check PostgreSQL logs
kubectl logs my-postgres-postgresql-0 -n test

# Test connectivity from Django pod
kubectl exec -it <django-pod-name> -n test -- nslookup my-postgres-postgresql
```

#### 6. General Debugging Commands
```sh
# Check all resources in test namespace
kubectl get all -n test

# Check events for issues
kubectl get events -n test --sort-by='.lastTimestamp'

# Check pod details
kubectl describe pod <pod-name> -n test

# Check service details
kubectl describe svc <service-name> -n test

# Check PVC details
kubectl describe pvc <pvc-name> -n test
```

#### 7. ArgoCD Application Issues
```sh
# Check ArgoCD application status
kubectl get applications -n argocd

# Check application details
kubectl describe application django-ledger-app -n argocd
kubectl describe application my-postgres -n argocd

# Check ArgoCD logs
kubectl logs -l app.kubernetes.io/name=argocd-application-controller -n argocd
```

#### 8. Network and DNS Issues
```sh
# Test DNS resolution from within a pod
kubectl run test-dns --image=busybox --rm -it --restart=Never -- nslookup my-postgres-postgresql

# Check if services are accessible
kubectl run test-connectivity --image=busybox --rm -it --restart=Never -- wget -O- http://my-postgres-postgresql:5432
```

---

## Notes
- **CRITICAL:** Always deploy supporting resources (PVs, PVCs, ConfigMaps, Secrets) BEFORE deploying ArgoCD applications.
- With this approach, you do not need to manually run `helm install` or `helm upgrade`.
- All configuration is stored in Git. To make changes, you update the files in your repository, commit, and push. ArgoCD will automatically sync the changes to your cluster.
- **Supporting resources (ConfigMap, Secret, PVCs) must exist in the same namespace as your app.**
- For production, ensure you manage secrets securely (e.g., with a tool like Vault or Sealed Secrets) instead of putting plain text passwords in the YAML files.
- If you see errors about missing ConfigMaps, Secrets, or PVCs, ensure they are created in the correct namespace.
- You can template these resources into your Helm chart for full GitOps management.

---

## References
- [Helm Docs](https://helm.sh/docs/)
- [Bitnami PostgreSQL Chart](https://artifacthub.io/packages/helm/bitnami/postgresql)
- [ArgoCD Docs](https://argo-cd.readthedocs.io/)

USE THES COMMANDS TO CLEAN UP 
```sh
kubectl logs my-postgres-postgresql-0 -n test --tail=50

kubectl delete app my-postgres -n argocd

kubectl get pods,pvc,pv -n test -w

kubectl delete all --all -n default

kubectl delete pvc --all -n default

kubectl delete configmap django-app-config -n default

kubectl delete secret django-app-secret -n default

kubectl delete pv postgres-pv

kubectl delete pv django-static-pv

sleep 30 && kubectl get pods -n test


#Step 3: Deploy Supporting Resources to Test Namespace 

# Step 3a: Apply PersistentVolumes (PVs)
kubectl apply -f django-ledger-starter/manifests/pv-postgres.yml -n test

#Step 3b: Apply PersistentVolumeClaims (PVCs)
kubectl apply -f django-ledger-starter/manifests/pv-static.yml -n test

kubectl apply -f django-ledger-starter/manifests/pvc-static.yml -n test

#Apply Secret 
kubectl apply -f django-ledger-starter/manifests/pvc-static.yml -n test

#Step 3c: Apply ConfigMap and Secret
kubectl apply -f django-ledger-starter/manifests/configmap.yml -n test

kubectl apply -f django-ledger-starter/manifests/secret.yml -n test
#kubectl apply -f django-ledger-starter/manifests/secret.yml -n test
kubectl get configmap,secret -n test

#Step 4: Deploy with ArgoCD
kubectl apply -f postgres-application.yaml -n argocd

kubectl apply -f django-application.yaml -n argocd

#Step 5: Monitor the Deployment
kubectl get pods -n test
kubectl get svc -n test

#Step 6: Run Database Migration Job

kubectl apply -f django-ledger-starter/manifests/django-migration-job.yml -n test

kubectl get jobs -n test

kubectl get jobs -n test

✅ Database connection established
✅ All Django migrations applied (including django_ledger migrations)
✅ Static files collected (148 files)


#Step 7: Test the Application
kubectl port-forward svc/django-ledger-app-django-ledger-chart 8000:8000 -n test

#Create superuser and login into the admin
kubectl exec -it -n test django-ledger-app-django-ledger-chart-5f575c6499-hj7dn -- bash

#Apply Nginx Application and service manifest 
kubectl apply -f django-ledger-starter/manifests/nginx-deployment.yml -n test

kubectl apply -f django-ledger-starter/manifests/nginx-service.yml -n test

# Test the app by running 
kubectl port-forward svc/nginx-service 8080:80 -n test