# Setting Up Kaniko for Docker Builds in Woodpecker CI

## Overview

Kaniko is a secure alternative to the Docker plugin that builds container images without requiring privileged access or Docker daemon access.

## Benefits

- ✅ **No privileged plugins required** - More secure
- ✅ **No Docker daemon needed** - Runs in user namespace
- ✅ **Same functionality** - Builds and pushes Docker images
- ✅ **Better isolation** - Reduced security risk

## Setup Steps

### 1. Remove Privileged Plugins (Already Done)

The `WOODPECKER_PLUGINS_PRIVILEGED` setting has been removed from the Woodpecker configuration.

### 2. Create GitHub Personal Access Token

1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate a new token with these scopes:
   - `write:packages` - To push images to GitHub Container Registry
   - `read:packages` - To pull images (optional)
3. Copy the token (you'll need it for step 3)

### 3. Add Secrets to Woodpecker

1. Go to your repository in Woodpecker UI: `https://woodpecker.sk8server.me`
2. Click on your repository → Settings → Secrets
3. Add these secrets:
   - **Name**: `docker_username`
     - **Value**: Your GitHub username (e.g., `ColeCoffman`)
   - **Name**: `docker_password`
     - **Value**: Your GitHub Personal Access Token from step 2

### 4. Update Your Pipeline

Use the Kaniko-based pipeline from `docs/WOODPECKER_PIPELINE_KANIKO.yml`:

```yaml
docker-build:
  image: gcr.io/kaniko-project/executor:latest
  commands:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"ghcr.io\":{\"auth\":\"$(echo -n $${DOCKER_USERNAME}:$${DOCKER_PASSWORD} | base64)\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor
        --context=.
        --dockerfile=Dockerfile
        --destination=ghcr.io/your-username/your-nextjs-app:${CI_COMMIT_SHA:0:7}
        --destination=ghcr.io/your-username/your-nextjs-app:latest
        --cache=true
        --cache-ttl=24h
        --snapshot-mode=redo
        --use-new-run
  secrets:
    - docker_username
    - docker_password
  depends_on:
    - build
  when:
    event:
      - push
    branch:
      - main
      - master
```

### 5. Update Image Repository

Replace these placeholders in your pipeline:
- `your-username` → Your GitHub username
- `your-nextjs-app` → Your Next.js app name

For example, if your GitHub username is `ColeCoffman` and your app is `my-nextjs-app`:
```yaml
--destination=ghcr.io/ColeCoffman/my-nextjs-app:${CI_COMMIT_SHA:0:7}
--destination=ghcr.io/ColeCoffman/my-nextjs-app:latest
```

### 6. Ensure Dockerfile Exists

Make sure you have a `Dockerfile` in the root of your Next.js repository. If you don't have one, here's a basic example:

```dockerfile
FROM node:20-alpine AS base

# Install dependencies only when needed
FROM base AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci

# Rebuild the source code only when needed
FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production image
FROM base AS runner
WORKDIR /app
ENV NODE_ENV production
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static

USER nextjs
EXPOSE 3000
ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

**Note**: For Next.js standalone output, add this to your `next.config.js`:
```javascript
module.exports = {
  output: 'standalone',
}
```

## Kaniko Options Explained

- `--context=.` - Build context (current directory)
- `--dockerfile=Dockerfile` - Path to Dockerfile
- `--destination` - Where to push the image (can specify multiple)
- `--cache=true` - Enable layer caching
- `--cache-ttl=24h` - Cache expiration time
- `--snapshot-mode=redo` - Snapshot mode for better performance
- `--use-new-run` - Use new run implementation

## Testing

1. Commit and push your `.woodpecker.yml` with Kaniko configuration
2. Check the pipeline in Woodpecker UI
3. The build should complete without requiring privileged plugins
4. Verify the image was pushed to `ghcr.io/your-username/your-app`

## Troubleshooting

### Authentication Errors
- Verify `docker_username` and `docker_password` secrets are set correctly
- Ensure GitHub token has `write:packages` scope
- Check that the token hasn't expired

### Build Failures
- Ensure `Dockerfile` exists in repository root
- Check that Dockerfile syntax is correct
- Verify build context includes all necessary files

### Permission Errors
- Kaniko runs as non-root by default (secure)
- If you need root, add `--run-as-user=0` (not recommended)

## Next Steps

After Kaniko is working, you can:
1. Set up Kubernetes deployment to use the built images
2. Configure image pull secrets for your cluster
3. Create HelmRelease or Kubernetes manifests for your Next.js app

