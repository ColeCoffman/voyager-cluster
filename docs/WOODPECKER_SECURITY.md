# Woodpecker CI Security Considerations

## Privileged Plugins - Security Risks

### What are Privileged Plugins?

Privileged plugins in Woodpecker CI have elevated permissions, typically allowing them to:
- Access the Docker daemon (for building/pushing images)
- Mount host filesystems
- Access sensitive system resources
- Run with elevated capabilities

### Security Risks

1. **Container Escape**: Privileged plugins run with elevated permissions that could potentially escape container isolation
2. **Host Access**: Access to Docker daemon means potential access to host system
3. **Supply Chain Attacks**: If a plugin is compromised, it has more power to cause damage
4. **Pipeline Injection**: Malicious code in your repository could abuse privileged plugins

### The Docker Plugin Specifically

The `plugins/docker` plugin needs privileged access because it:
- Connects to the Docker daemon (usually via `/var/run/docker.sock`)
- Builds Docker images
- Pushes images to registries

This requires access that could be used maliciously if your repository is compromised.

## Alternatives to Privileged Docker Plugin

### Option 1: Use Kaniko (Recommended - More Secure)

Kaniko builds Docker images without requiring Docker daemon access:

```yaml
docker-build:
  image: gcr.io/kaniko-project/executor:latest
  commands:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"ghcr.io\":{\"auth\":\"$(echo -n $DOCKER_USERNAME:$DOCKER_PASSWORD | base64)\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor
        --context=.
        --dockerfile=Dockerfile
        --destination=ghcr.io/your-username/your-app:${CI_COMMIT_SHA:0:7}
        --destination=ghcr.io/your-username/your-app:latest
  secrets:
    - docker_username
    - docker_password
  when:
    event:
      - push
    branch:
      - main
```

**Benefits:**
- No privileged access required
- Runs in user namespace
- More secure container isolation
- Same functionality as Docker plugin

### Option 2: Use Buildah

Buildah is another rootless container build tool:

```yaml
docker-build:
  image: quay.io/buildah/stable:latest
  commands:
    - buildah login --username $DOCKER_USERNAME --password $DOCKER_PASSWORD ghcr.io
    - buildah bud -t ghcr.io/your-username/your-app:${CI_COMMIT_SHA:0:7} .
    - buildah push ghcr.io/your-username/your-app:${CI_COMMIT_SHA:0:7}
  secrets:
    - docker_username
    - docker_password
```

### Option 3: Use Trusted Repositories Only

If you must use the Docker plugin:
1. Only enable privileged plugins for trusted repositories
2. Use repository-level settings instead of global
3. Limit which repositories can use privileged plugins
4. Review all pipeline code before merging

## Best Practices

### If Using Privileged Plugins:

1. **Limit Scope**: Only enable for specific repositories that need it
2. **Code Review**: Always review pipeline code before merging
3. **Principle of Least Privilege**: Only grant privileges to what's necessary
4. **Monitor**: Watch for unusual activity in pipelines
5. **Isolate**: Run Woodpecker agents in a separate namespace with limited RBAC

### General Security:

1. **Secrets Management**: Never commit secrets, always use Woodpecker secrets
2. **Repository Trust**: Only trust repositories you control
3. **Regular Updates**: Keep Woodpecker and plugins updated
4. **Network Policies**: Use Kubernetes network policies to limit agent access
5. **RBAC**: Limit what Woodpecker agents can do in your cluster

## Recommendation

**Use Kaniko instead of the privileged Docker plugin** - it provides the same functionality without the security risks.

