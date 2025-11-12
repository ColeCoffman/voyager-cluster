# Jenkins Alternative Setup

If you prefer Jenkins over Woodpecker CI, here's how to set it up.

## Why Choose Jenkins?

- ✅ More established and mature
- ✅ Huge plugin ecosystem
- ✅ More enterprise features
- ✅ Better for complex pipelines
- ⚠️ Heavier resource usage
- ⚠️ More complex setup

## Quick Comparison

| Feature | Woodpecker CI | Jenkins |
|---------|---------------|---------|
| Setup Complexity | ⭐ Easy | ⭐⭐⭐ Complex |
| Resource Usage | Light (~500MB) | Heavy (~2GB+) |
| GUI Quality | Modern | Classic |
| Learning Curve | Easy | Steeper |
| Kubernetes Native | ✅ Yes | ⚠️ Via plugin |
| Best For | Small/Medium teams | Enterprise |

## Jenkins Setup

Jenkins would be deployed similarly to Woodpecker, but requires more configuration.

### Recommended: Use Jenkins Operator

The Jenkins Operator makes it easier to manage Jenkins on Kubernetes.

## Recommendation

**For your use case (Next.js app, PR previews, beginner-friendly):**
- **Choose Woodpecker CI** - It's simpler, lighter, and perfect for what you need.

**Choose Jenkins if:**
- You need enterprise features
- You have complex build requirements
- You're already familiar with Jenkins
- You need specific Jenkins plugins

