# Quick Start: CI/CD for Next.js App

## 🎯 What You're Getting

A complete CI/CD setup with:
- **Web GUI** (Woodpecker CI) to see builds and logs
- **Auto-deploy** when you push to main branch
- **PR previews** - automatic preview URLs for pull requests
- **Simple setup** - everything runs in your cluster

## 📋 Quick Setup (5 Steps)

### 1. Create GitHub OAuth App

1. Go to: https://github.com/settings/developers
2. Click "New OAuth App"
3. Fill in:
   - Name: `Woodpecker CI`
   - Homepage: `https://woodpecker.sk8server.me`
   - Callback: `https://woodpecker.sk8server.me/authorize`
4. Save the **Client ID** and **Client Secret**

### 2. Generate Secrets

```bash
# Generate agent secret
openssl rand -hex 32
```

### 3. Configure Secret File

Edit `kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml`:
- Replace `your-github-client-id` with your Client ID
- Replace `your-github-client-secret` with your Client Secret
- Replace `your-random-secret-here` with the generated secret

Then encrypt:
```bash
sops --encrypt --in-place kubernetes/apps/ci-cd/woodpecker/app/secret.sops.yaml
```

### 4. Deploy

```bash
git add kubernetes/apps/ci-cd/
git commit -m "feat: add Woodpecker CI"
git push
```

Flux will automatically deploy it!

### 5. Access & Configure

1. Open: `https://woodpecker.sk8server.me`
2. Login with GitHub
3. Activate your Next.js repository
4. Add `.woodpecker.yml` to your Next.js repo (see example in docs)

## 🚀 That's It!

Now when you:
- **Push to main** → Builds and deploys automatically
- **Open a PR** → Creates preview environment at `pr-123.your-app.sk8server.me`

## 📚 More Details

- Full setup guide: `docs/WOODPECKER_SETUP_GUIDE.md`
- Next.js example: `docs/NEXTJS_WOODPECKER_EXAMPLE.md`
- General CI/CD info: `docs/CI_CD_SETUP.md`

## 🆘 Need Help?

Check the troubleshooting section in `docs/WOODPECKER_SETUP_GUIDE.md`

