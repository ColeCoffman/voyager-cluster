# Woodpecker CI Pipeline for Next.js

This guide shows how to create a Woodpecker CI pipeline for your Next.js application.

## Basic Pipeline Structure

Create a `.woodpecker.yml` file in the root of your Next.js repository:

```yaml
when:
  event:
    - push
    - pull_request

steps:
  # Install dependencies
  install:
    image: node:20-alpine
    commands:
      - npm ci

  # Run linting
  lint:
    image: node:20-alpine
    commands:
      - npm run lint
    depends_on:
      - install

  # Run tests
  test:
    image: node:20-alpine
    commands:
      - npm run test
    depends_on:
      - install

  # Build the application
  build:
    image: node:20-alpine
    commands:
      - npm run build
    depends_on:
      - install
    when:
      event:
        - push
        - pull_request

  # Build Docker image (if deploying)
  docker-build:
    image: plugins/docker
    settings:
      repo: ghcr.io/your-username/your-nextjs-app
      tags:
        - latest
        - ${CI_COMMIT_SHA}
      username:
        from_secret: docker_username
      password:
        from_secret: docker_password
    depends_on:
      - build
    when:
      event:
        - push
        branch:
          - main
          - master

  # Deploy to Kubernetes (optional)
  deploy:
    image: bitnami/kubectl:latest
    commands:
      - kubectl set image deployment/your-app app=ghcr.io/your-username/your-nextjs-app:${CI_COMMIT_SHA}
    secrets:
      - kubeconfig
    depends_on:
      - docker-build
    when:
      event:
        - push
      branch:
        - main
        - master
```

## Quick Start Template

For a simple Next.js pipeline, use this minimal template:

```yaml
when:
  event:
    - push
    - pull_request

steps:
  install:
    image: node:20-alpine
    commands:
      - npm ci

  build:
    image: node:20-alpine
    commands:
      - npm run build
    depends_on:
      - install

  test:
    image: node:20-alpine
    commands:
      - npm test
    depends_on:
      - install
```

## Next Steps

1. **Create `.woodpecker.yml`** in your Next.js repository root
2. **Commit and push** the file
3. **Woodpecker will automatically detect** the pipeline and run it
4. **View results** in the Woodpecker UI at `https://woodpecker.sk8server.me`

## Advanced: Docker Build & Deploy

If you want to build and deploy Docker images:

1. **Add Dockerfile** to your Next.js project (if not already present)
2. **Set up secrets** in Woodpecker:
   - Go to your repository settings in Woodpecker
   - Add secrets for:
     - `docker_username` - Your GitHub username or Docker Hub username
     - `docker_password` - Your GitHub Personal Access Token (with `write:packages` scope) or Docker Hub password
3. **Update pipeline** to include Docker build step

## Example Dockerfile for Next.js

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

## Tips

- Use `npm ci` instead of `npm install` for faster, reliable builds
- Cache `node_modules` if builds are slow (Woodpecker supports volume caching)
- Run tests before building to fail fast
- Use matrix builds for testing multiple Node.js versions
- Set up branch protection rules in GitHub to require passing pipelines

