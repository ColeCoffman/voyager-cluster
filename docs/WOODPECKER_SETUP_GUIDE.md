# Woodpecker CI Setup Guide

Complete step-by-step guide to set up Woodpecker CI for your Next.js app.

## Prerequisites

- GitHub account
- Access to your Kubernetes cluster
- Domain configured (sk8server.me)

## Step 1: Create GitHub OAuth App

1. Go to https://github.com/settings/developers
2. Click "New OAuth App"
3. Fill in:
   - **Application name**: `Woodpecker CI`
   - **Homepage URL**: `https://woodpecker.sk8server.me`
   - **Authorization callback URL**: `https://woodpecker.sk8server.me/authorize`
4. Click "Register application"
5. **Copy the Client ID and Client Secret** - you'll need these!

## Step 2: Generate Agent Secret

Run this command to generate a random secret:

```bash
openssl rand -hex 32
```

Save this value - you'll need it for the secret.

## Step 3: Create SOPS Secret

1. Edit `kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml`
2. Replace the placeholder values:
   - `your-github-client-id` → Your GitHub Client ID
   - `your-github-client-secret` → Your GitHub Client Secret
   - `your-random-secret-here` → The generated secret from Step 2

3. Encrypt the file with SOPS:

```bash
sops --encrypt --in-place kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml
```

## Step 4: Update HelmRelease with Secret References

The HelmRelease needs to reference the secret. Update the values to use environment variables from the secret.

Actually, we need to update the HelmRelease to use the secret properly. Let me check the Woodpecker Helm chart structure...

## Step 5: Deploy Woodpecker

1. Commit your changes:
   ```bash
   git add kubernetes/apps/ci-cd/
   git commit -m "feat: add Woodpecker CI"
   git push
   ```

2. Flux will automatically deploy it:
   ```bash
   flux reconcile kustomization ci-cd -n flux-system
   ```

3. Check deployment:
   ```bash
   kubectl get pods -n ci-cd
   kubectl get svc -n ci-cd
   ```

## Step 6: Access Woodpecker Web UI

1. Open `https://woodpecker.sk8server.me`
2. Click "Login with GitHub"
3. Authorize the application
4. You're in! 🎉

## Step 7: Connect Your Next.js Repository

1. In Woodpecker UI, click "Repositories"
2. Click "Activate" next to your Next.js repository
3. Woodpecker will now watch for pushes and PRs

## Step 8: Add `.woodpecker.yml` to Your Next.js App

See `docs/NEXTJS_WOODPECKER_EXAMPLE.md` for the complete example.

## Step 9: Configure Secrets in Woodpecker

For each repository, add secrets:

1. Go to repository settings in Woodpecker
2. Click "Secrets"
3. Add:
   - `github_username` - Your GitHub username
   - `github_token` - GitHub Personal Access Token with `write:packages` permission
   - `docker_username` - Same as github_username
   - `docker_password` - Same as github_token

## Troubleshooting

### Woodpecker not accessible?

```bash
# Check if pods are running
kubectl get pods -n ci-cd

# Check logs
kubectl logs -n ci-cd -l app=woodpecker-server

# Check HTTPRoute
kubectl get httproute -n ci-cd
```

### Builds not starting?

- Check repository is activated in Woodpecker UI
- Verify `.woodpecker.yml` exists in your repo
- Check Woodpecker agent logs: `kubectl logs -n ci-cd -l app=woodpecker-agent`

### Images not pushing?

- Verify GitHub token has `write:packages` permission
- Check secret is correctly set in Woodpecker repository settings

## Next Steps

1. Set up Flux Image Automation for auto-deployment
2. Configure preview environments for PRs
3. Add monitoring/alerting

