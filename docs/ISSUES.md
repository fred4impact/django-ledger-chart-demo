# Common Pod Issues and Resolutions

## 1. CrashLoopBackOff Issues

### the cause.
- Pods continuously restarting
- Status showing `CrashLoopBackOff`
- Application logs showing startup errors

### Common Causes and Solutions

#### Database Connection Issues
```bash
# Check if PostgreSQL pod is running
kubectl get pods | grep postgres

# Verify database credentials
kubectl get secret django-app-secret -o yaml

# Check database connection logs
kubectl logs -l app.kubernetes.io/name=django-ledger
```

**Resolution**: 
- Ensure PostgreSQL pod is running before Django pod starts
- Verify DATABASE_URL in values.yaml matches the service name
- Check if database credentials are correct

#### Missing Environment Variables
```bash
# Check pod environment variables
kubectl describe pod <pod-name>

# Verify ConfigMap
kubectl get configmap django-app-config -o yaml
```

**Resolution**:
- Add missing environment variables to values.yaml
- Ensure all required Django settings are configured

## 2. ImagePullBackOff Errors

### Symptoms
- Pods stuck in `ImagePullBackOff` state
- Unable to pull Docker image

### Common Causes and Solutions

#### Private Registry Authentication
```bash
# Create docker-registry secret
kubectl create secret docker-registry regcred \
  --docker-server=<your-registry-server> \
  --docker-username=<your-username> \
  --docker-password=<your-password> \
  --docker-email=<your-email>

# Add imagePullSecrets to deployment
kubectl patch deployment django-ledger-django -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"regcred"}]}}}}'
```

**Resolution**:
- Create Kubernetes secret for registry authentication
- Add imagePullSecrets to deployment
- Verify image name and tag in values.yaml

#### Minikube Image Access
```bash
# Load image into Minikube
minikube image load runtesting/django-ledger:1.0

# Verify image exists in Minikube
minikube ssh "docker images | grep django-ledger"
```

**Resolution**:
- Load image into Minikube's Docker daemon
- Use local image repository
- Verify image exists in Minikube

## 3. PVC Binding Issues

### Symptoms
- PersistentVolumeClaims stuck in `Pending` state
- Storage not being provisioned

### Common Causes and Solutions

#### Storage Class Issues
```bash
# Check available storage classes
kubectl get storageclass

# Verify PVC configuration
kubectl get pvc
kubectl describe pvc django-ledger-postgres
```

**Resolution**:
- Ensure correct storage class is specified
- Verify storage size requirements
- Check if Minikube has enough storage

#### Permission Issues
```bash
# Check PVC events
kubectl describe pvc django-ledger-postgres

# Verify pod security context
kubectl describe pod <pod-name>
```

**Resolution**:
- Add appropriate security context to pods
- Verify volume mount permissions
- Check if service account has necessary permissions

## 4. Database Connection Issues

### Symptoms
- Application unable to connect to database
- Database connection timeouts
- Authentication failures

### Common Causes and Solutions

#### Service Name Resolution
```bash
# Verify service exists
kubectl get svc postgres-service

# Check service endpoints
kubectl get endpoints postgres-service

# Test DNS resolution
kubectl run -it --rm --restart=Never busybox --image=busybox:1.28 -- nslookup postgres-service
```

**Resolution**:
- Ensure service name matches DATABASE_URL
- Verify service is in the same namespace
- Check if service ports are correctly configured

#### Database Credentials
```bash
# Verify secret exists
kubectl get secret django-app-secret

# Check if secret is mounted correctly
kubectl describe pod <pod-name>

# Verify database user exists
kubectl exec -it <postgres-pod> -- psql -U postgres -c "\du"
```

**Resolution**:
- Ensure database user exists
- Verify password in secret matches database
- Check if database exists and is accessible

## 5. Secret Management Issues

### Symptoms
- Application unable to read secrets
- Authentication failures
- Configuration errors

### Common Causes and Solutions

#### Secret Creation
```bash
# Create secret from literal values
kubectl create secret generic django-app-secret \
  --from-literal=DB_NAME=django_ledger \
  --from-literal=DB_USERNAME=postgres \
  --from-literal=DB_PASSWORD=postgres \
  --from-literal=DATABASE_URL=postgres://postgres:postgres@postgres-service:5432/django_ledger

# Verify secret
kubectl get secret django-app-secret -o yaml
```

**Resolution**:
- Create secrets using kubectl or Helm
- Ensure secrets are in the correct namespace
- Verify secret keys match application expectations

#### Secret Mounting
```bash
# Check if secret is mounted
kubectl describe pod <pod-name>

# Verify secret volume
kubectl get pod <pod-name> -o yaml | grep -A 5 volumes
```

**Resolution**:
- Ensure secret is properly mounted in pod
- Verify volume mount path
- Check secret key references in deployment

## Best Practices

1. **Always check logs first**:
```bash
kubectl logs <pod-name>
kubectl describe pod <pod-name>
```

2. **Verify dependencies**:
- Database pod is running
- Services are properly configured
- Secrets and ConfigMaps exist

3. **Use proper initialization order**:
- Database should be ready before application
- Use init containers if needed
- Implement proper health checks

4. **Monitor resources**:
```bash
kubectl top pods
kubectl top nodes
```

5. **Regular maintenance**:
- Clean up old pods and deployments
- Monitor storage usage
- Check for resource limits 