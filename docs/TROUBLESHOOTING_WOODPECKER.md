# Troubleshooting Woodpecker CI

## Issue: Can't Access Woodpecker URL

If you can't access `https://woodpecker.sk8server.me`, follow these steps:

### Step 1: Check if Resources are Deployed

```bash
# Check namespace exists
kubectl get namespace ci-cd

# Check pods
kubectl get pods -n ci-cd

# Check HelmRelease status
kubectl get helmrelease -n ci-cd
kubectl describe helmrelease woodpecker -n ci-cd

# Check services
kubectl get svc -n ci-cd

# Check HTTPRoute
kubectl get httproute -n ci-cd
kubectl describe httproute woodpecker -n ci-cd
```

### Step 2: Check Pod Logs

```bash
# Check server logs
kubectl logs -n ci-cd -l app=woodpecker-server

# Check agent logs
kubectl logs -n ci-cd -l app=woodpecker-agent

# Check for errors
kubectl get events -n ci-cd --sort-by='.metadata.creationTimestamp'
```

### Step 3: Verify Flux Sync

```bash
# Check if Flux has synced
flux get kustomizations -n ci-cd
flux get helmreleases -n ci-cd

# Force reconcile
flux reconcile kustomization ci-cd -n flux-system
flux reconcile helmrelease woodpecker -n ci-cd
```

### Step 4: Common Issues

#### Issue: HelmRelease Not Ready

**Symptoms:**
- `kubectl get helmrelease` shows `Not Ready`
- Pods not created

**Solutions:**
1. Check HelmRepository:
   ```bash
   kubectl get helmrepository -n ci-cd
   kubectl describe helmrepository woodpecker -n ci-cd
   ```

2. Check if chart can be fetched:
   ```bash
   helm repo add woodpecker https://charts.woodpecker-ci.org
   helm repo update
   helm search repo woodpecker
   ```

3. Check HelmRelease events:
   ```bash
   kubectl describe helmrelease woodpecker -n ci-cd
   ```

#### Issue: Secret Not Configured

**Symptoms:**
- Pods crash or won't start
- Logs show "missing environment variable"

**Solutions:**
1. Verify secret exists:
   ```bash
   kubectl get secret woodpecker-secrets -n ci-cd
   ```

2. Check if secret is decrypted (if using SOPS):
   ```bash
   kubectl get secret woodpecker-secrets -n ci-cd -o yaml
   ```

3. Re-encrypt secret if needed:
   ```bash
   sops --encrypt --in-place kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml
   ```

#### Issue: Service Not Found

**Symptoms:**
- HTTPRoute shows errors
- Can't connect to service

**Solutions:**
1. Check service name matches HTTPRoute:
   ```bash
   kubectl get svc -n ci-cd
   # Should see: woodpecker-server
   ```

2. Verify service port:
   ```bash
   kubectl get svc woodpecker-server -n ci-cd -o yaml
   # Should expose port 8000
   ```

3. Update HTTPRoute if service name/port is different

#### Issue: HTTPRoute Not Working

**Symptoms:**
- Service exists but can't access via URL

**Solutions:**
1. Check HTTPRoute status:
   ```bash
   kubectl describe httproute woodpecker -n ci-cd
   ```

2. Verify Gateway is accessible:
   ```bash
   kubectl get gateway -n network
   ```

3. Check DNS:
   ```bash
   dig woodpecker.sk8server.me
   # Should resolve to your gateway IP
   ```

4. Test service directly:
   ```bash
   kubectl port-forward -n ci-cd svc/woodpecker-server 8000:8000
   # Then visit http://localhost:8000
   ```

### Step 5: Manual Deployment Test

If Flux isn't working, try manual deployment:

```bash
# Create namespace
kubectl create namespace ci-cd

# Apply resources manually
kubectl apply -f kubernetes/apps/ci-cd/woodpecker/app/namespace.yaml
kubectl apply -f kubernetes/apps/ci-cd/woodpecker/app/helmrepository.yaml
kubectl apply -f kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml  # After decrypting
kubectl apply -f kubernetes/apps/ci-cd/woodpecker/app/helmrelease.yaml
kubectl apply -f kubernetes/apps/ci-cd/woodpecker/app/httproute.yaml
```

### Step 6: Check Configuration Files

Verify these files are correct:

1. **Secret file** (`secret.sops.yaml`):
   - Must have actual values (not placeholders)
   - Must be encrypted with SOPS
   - Values must match GitHub OAuth app

2. **HelmRelease** (`helmrelease.yaml`):
   - Chart name should be correct
   - Values should be valid YAML
   - Service port should match HTTPRoute

3. **HTTPRoute** (`httproute.yaml`):
   - Service name must match actual service
   - Port must match service port
   - Hostname must be correct

### Step 7: Get Detailed Debug Info

```bash
# Get all resources in namespace
kubectl get all -n ci-cd

# Get all events
kubectl get events -n ci-cd --sort-by='.metadata.creationTimestamp'

# Check Flux logs
kubectl logs -n flux-system -l app=helm-controller --tail=100
kubectl logs -n flux-system -l app=kustomize-controller --tail=100
```

## Still Not Working?

1. **Check the Quick Start guide** - Make sure you completed all steps
2. **Verify GitHub OAuth app** - Ensure callback URL is correct
3. **Check cluster connectivity** - Ensure kubectl can connect
4. **Review Flux documentation** - Check Flux is working correctly

## Getting Help

If you're still stuck, collect this information:

```bash
# Save debug info
kubectl get all -n ci-cd -o yaml > woodpecker-debug.yaml
kubectl describe helmrelease woodpecker -n ci-cd > woodpecker-helmrelease.txt
kubectl logs -n ci-cd -l app=woodpecker-server --tail=100 > woodpecker-server-logs.txt
```

Then check:
- Are pods running?
- Are services created?
- Is HTTPRoute configured?
- Are there any error messages in logs?

