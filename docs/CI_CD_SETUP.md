# CI/CD Setup for Next.js App with GUI

This guide explains how to set up a GUI-based CI/CD pipeline for your Next.js application on your Kubernetes cluster.

## 🎯 What You'll Get

- **Web GUI** to see builds, logs, and pipeline status
- **Auto-deploy on git push** to main branch
- **PR previews** - automatic preview environments for pull requests
- **Build history** - see all your builds in one place
- **Simple setup** - no complex configuration needed

## 🏆 Recommended Solution: Woodpecker CI

**Why Woodpecker CI?**
- ✅ Modern, lightweight, and Kubernetes-native
- ✅ Beautiful web GUI (similar to Drone CI)
- ✅ Simple YAML-based pipelines (like GitHub Actions)
- ✅ Runs entirely in your cluster
- ✅ Great for beginners
- ✅ Supports PR previews out of the box
- ✅ Free and open source

**Alternative: Jenkins**
- More established, but heavier and more complex
- Better if you need enterprise features
- More plugins available

## Architecture Overview

```
┌─────────────┐
│  Git Push   │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Woodpecker CI   │  ← GUI here! (web interface)
│  (Build Server)  │
└──────┬──────────┘
       │
       ├─► Build Next.js app
       ├─► Create Docker image
       └─► Push to registry
              │
              ▼
       ┌──────────────┐
       │  Flux        │  ← Auto-deploys from git
       │  (GitOps)    │
       └──────┬───────┘
              │
              ▼
       ┌──────────────┐
       │  Kubernetes  │
       │  (Your App)  │
       └──────────────┘
```

## How It Works

1. **You push code** → Woodpecker CI detects the push
2. **Woodpecker builds** → Creates Docker image of your Next.js app
3. **Image pushed** → To GitHub Container Registry (or your registry)
4. **Flux watches** → Detects new image and auto-deploys
5. **App live** → Your Next.js app is running!

For PRs:
- Same process, but creates a preview environment
- Preview URL: `pr-123.your-app.sk8server.me`
- Auto-cleanup when PR is closed

## Setup Steps

### Step 1: Install Woodpecker CI

Woodpecker CI will be deployed to your cluster via Flux (just like your other apps).

### Step 2: Configure Your Next.js App

Add a `.woodpecker.yml` file to your Next.js app repository with build instructions.

### Step 3: Connect Woodpecker to GitHub

Authorize Woodpecker to access your GitHub repositories.

### Step 4: Configure Flux for Auto-Deployment

Set up Flux Image Automation to watch for new images and deploy automatically.

## What You'll See in the GUI

- **Dashboard** - Overview of all builds
- **Pipeline View** - Step-by-step build progress
- **Logs** - Real-time build logs
- **Build History** - Past builds and their status
- **Repository Settings** - Configure build triggers

## Next Steps

See the implementation files:
- `kubernetes/apps/ci-cd/woodpecker/` - Woodpecker CI deployment
- Example `.woodpecker.yml` for Next.js
- Flux Image Automation configuration

