# Next.js App Setup with Woodpecker CI

This guide shows you how to configure your Next.js app to work with Woodpecker CI.

## Step 1: Create `.woodpecker.yml` in Your Next.js App

Create this file in the root of your Next.js repository:

```yaml
# .woodpecker.yml
when:
  branch:
    - main
    - develop
  event:
    - push
    - pull_request

steps:
  # Build and test
  build:
    image: node:20-alpine
    commands:
      - npm ci
      - npm run build
      - npm test || true  # Run tests if you have them

  # Build Docker image
  docker:
    image: plugins/docker
    settings:
      registry: ghcr.io
      username:
        from_secret: github_username
      password:
        from_secret: github_token
      repo: ${CI_REPO_OWNER}/${CI_REPO_NAME}
      tags:
        - latest
        - ${CI_COMMIT_SHA:0:7}
        - ${CI_BUILD_NUMBER}
      # For PRs, tag with PR number
      tags_auto:
        - pr-${CI_PULL_REQUEST}
      when:
        event:
          - pull_request
      tags:
        - pr-${CI_PULL_REQUEST}
        - pr-${CI_PULL_REQUEST}-${CI_COMMIT_SHA:0:7}
    when:
      event:
        - pull_request

  # For main branch, also tag as latest
  docker-main:
    image: plugins/docker
    settings:
      registry: ghcr.io
      username:
        from_secret: github_username
      password:
        from_secret: github_token
      repo: ${CI_REPO_OWNER}/${CI_REPO_NAME}
      tags:
        - latest
        - ${CI_COMMIT_SHA:0:7}
        - ${CI_BUILD_NUMBER}
    when:
      branch:
        - main
      event:
        - push

  # Deploy preview environment for PRs
  deploy-preview:
    image: bitnami/kubectl:latest
    commands:
      - |
        kubectl create namespace preview-pr-${CI_PULL_REQUEST} --dry-run=client -o yaml | kubectl apply -f -
        # Create deployment, service, and ingress for preview
        # (This would be handled by Flux in a GitOps approach)
    when:
      event:
        - pull_request
    secrets:
      - kubeconfig
```

## Step 2: Create Dockerfile for Your Next.js App

Create a `Dockerfile` in your Next.js app root:

```dockerfile
# Dockerfile
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

# Set environment variable for build
ENV NEXT_TELEMETRY_DISABLED 1
ENV NODE_ENV production

RUN npm run build

# Production image, copy all the files and run next
FROM base AS runner
WORKDIR /app

ENV NODE_ENV production
ENV NEXT_TELEMETRY_DISABLED 1

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

**Important:** Update your `next.config.js` to enable standalone output:

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone', // This is required for the Dockerfile
}

module.exports = nextConfig
```

## Step 3: Configure Secrets in Woodpecker

In the Woodpecker web UI:

1. Go to your repository settings
2. Add these secrets:
   - `github_username` - Your GitHub username
   - `github_token` - GitHub Personal Access Token with `write:packages` permission
   - `kubeconfig` - Your Kubernetes config (if using direct deployment)

## Step 4: Set Up Flux for Auto-Deployment

Create Flux resources to watch for new images and deploy automatically.

See: `kubernetes/apps/your-app/` for the Flux configuration.

## How It Works

1. **You push code** → Woodpecker detects it
2. **Woodpecker builds** → Runs `npm run build`
3. **Docker image created** → Tagged with commit SHA or PR number
4. **Image pushed** → To GitHub Container Registry
5. **Flux detects** → New image available
6. **Flux deploys** → Your app is live!

For PRs:
- Image tagged as `pr-123-abc1234`
- Preview environment created
- URL: `pr-123.your-app.sk8server.me`

## Viewing Builds in Woodpecker GUI

1. Open `https://woodpecker.sk8server.me`
2. Log in with GitHub
3. See all your repositories
4. Click on a repository to see builds
5. Click on a build to see logs and status

## Troubleshooting

- **Build fails?** Check logs in Woodpecker GUI
- **Image not deploying?** Check Flux logs: `flux logs -n flux-system`
- **Preview not working?** Verify HTTPRoute and namespace exist

