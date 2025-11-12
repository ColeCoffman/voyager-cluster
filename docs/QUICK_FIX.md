# Quick Fix: Woodpecker Not Accessible

## Immediate Steps to Debug

Run these commands to see what's happening:

```bash
# 1. Check if namespace exists
kubectl get namespace ci-cd

# 2. Check if pods are running
kubectl get pods -n ci-cd

# 3. Check HelmRelease status
kubectl get helmrelease -n ci-cd
kubectl describe helmrelease woodpecker -n ci-cd

# 4. Check services
kubectl get svc -n ci-cd

# 5. Check HTTPRoute
kubectl get httproute -n ci-cd
kubectl describe httproute woodpecker -n ci-cd

# 6. Check Flux sync status
flux get kustomizations -n ci-cd
flux get helmreleases -n ci-cd
```

## Common Issues & Fixes

### Issue 1: Secret Not Configured

**Problem:** The secret file still has placeholder values.

**Fix:**
1. Edit `kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml`
2. Replace placeholders with real values:
   - `your-github-client-id` → Your actual GitHub Client ID
   - `your-github-client-secret` → Your actual GitHub Client Secret
   - `your-random-secret-here` → Generated secret (run `openssl rand -hex 32`)
3. Encrypt with SOPS:
   ```bash
   sops --encrypt --in-place kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml
   ```
4. Commit and push

### Issue 2: HelmRelease Not Ready

**Problem:** Helm chart can't be fetched or installed.

**Fix:**
```bash
# Check HelmRepository
kubectl get helmrepository -n ci-cd
kubectl describe helmrepository woodpecker -n ci-cd

# Force Flux to reconcile
flux reconcile helmrelease woodpecker -n ci-cd
```

### Issue 3: Pods Not Starting

**Problem:** Pods are in CrashLoopBackOff or Pending.

**Fix:**
```bash
# Check pod logs
kubectl logs -n ci-cd -l app=woodpecker-server --tail=50

# Check events
kubectl get events -n ci-cd --sort-by='.metadata.creationTimestamp'
```

### Issue 4: Service Not Found

**Problem:** HTTPRoute can't find the service.

**Fix:**
1. Check service name:
   ```bash
   kubectl get svc -n ci-cd
   ```
2. Update HTTPRoute if service name is different (should be `woodpecker-server`)

### Issue 5: Flux Not Syncing

**Problem:** Changes pushed but Flux hasn't picked them up.

**Fix:**
```bash
# Force reconcile
flux reconcile kustomization ci-cd -n flux-system
flux reconcile source git flux-system -n flux-system
```

## What to Check First

1. **Did you configure the secret?** (Most common issue)
   - Secret must have real values, not placeholders
   - Must be encrypted with SOPS

2. **Is Flux syncing?**
   - Check `flux get kustomizations`
   - Check `flux get helmreleases`

3. **Are pods running?**
   - `kubectl get pods -n ci-cd`
   - Should see `woodpecker-server-*` and `woodpecker-agent-*` pods

4. **Is the service created?**
   - `kubectl get svc -n ci-cd`
   - Should see `woodpecker-server` service

5. **Is HTTPRoute configured?**
   - `kubectl get httproute -n ci-cd`
   - Should see `woodpecker` HTTPRoute

## Still Not Working?

See the full troubleshooting guide: `docs/TROUBLESHOOTING_WOODPECKER.md`

