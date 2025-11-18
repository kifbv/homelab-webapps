# homelab-webapps

AI-generated webapp source code for homelab dynamic application deployment.

## Overview

This repository contains source code for webapps dynamically generated and deployed via n8n workflows. Each application is built as a container image and deployed to the Kubernetes homelab cluster using GitOps principles.

## Repository Structure

```
homelab-webapps/
├── app-*/                  # Individual webapp directories
│   ├── Dockerfile          # Container image definition
│   ├── src/                # Application source code
│   └── README.md           # App-specific documentation
├── _example/               # Example webapp template
└── README.md               # This file
```

## Workflows

### New Webapp Creation (via n8n)

1. **Code Generation**: n8n workflow generates webapp code based on user requirements
2. **Push to Repository**: Generated code is committed and pushed to `app-*` directory
3. **Image Build**: n8n triggers Kaniko build in cluster with tag `v1.0.0`
4. **Image Push**: Kaniko pushes image to `registry.k8s-lab.dev`
5. **Manifest Creation**: n8n generates Kubernetes manifests in homelab repository
6. **Deployment**: Flux detects manifests and deploys to cluster

### Webapp Update (manual or automated)

1. **Code Modification**: User or n8n modifies code in `app-*` directory
2. **Commit**: Changes committed with conventional commit message (e.g., `fix:` or `feat:`)
3. **Webhook Trigger**: GitHub webhook notifies n8n of push event
4. **Version Calculation**: n8n parses commit message and queries registry for current version
5. **Semver Bump**: n8n calculates new version based on conventional commit type:
   - `fix:` → PATCH bump (v1.0.0 → v1.0.1)
   - `feat:` → MINOR bump (v1.0.1 → v1.1.0)
   - `BREAKING CHANGE:` → MAJOR bump (v1.1.0 → v2.0.0)
6. **Image Build**: n8n triggers Kaniko build with new version tag
7. **Image Push**: Kaniko pushes new image to registry
8. **Automatic Update**: Flux Image Automation detects new tag, updates manifest in homelab repo, and deploys

## Image Naming Convention

Images are built and tagged as:
```
registry.k8s-lab.dev/webapp/<app-name>:<version>
registry.k8s-lab.dev/webapp/<app-name>:latest
```

Example:
```
registry.k8s-lab.dev/webapp/app-example:v1.0.0
registry.k8s-lab.dev/webapp/app-example:v1.0.1
registry.k8s-lab.dev/webapp/app-example:latest
```

## Conventional Commits

This repository uses [Conventional Commits](https://www.conventionalcommits.org/) for automatic semantic versioning:

- `fix: <message>` - Patches a bug (PATCH version bump)
- `feat: <message>` - Introduces a new feature (MINOR version bump)
- `BREAKING CHANGE: <message>` - Breaking API change (MAJOR version bump)

Examples:
```bash
git commit -m "fix(auth): resolve token expiry issue"
git commit -m "feat(api): add user profile endpoint"
git commit -m "feat!: migrate to new authentication system

BREAKING CHANGE: OAuth tokens are no longer compatible"
```

## Adding a New Webapp

### Via n8n (Recommended)

Use the n8n workflow interface to generate and deploy webapps automatically.

### Manually

1. Create app directory:
   ```bash
   mkdir app-myapp
   cd app-myapp
   ```

2. Add your application code:
   ```
   app-myapp/
   ├── Dockerfile
   ├── src/
   │   └── (your code)
   └── README.md
   ```

3. Commit and push:
   ```bash
   git add app-myapp
   git commit -m "feat: add myapp application"
   git push
   ```

4. GitHub webhook triggers n8n, which builds the image

5. Create Kubernetes manifests in the homelab repository (or use n8n to generate them)

## Development Guidelines

### Dockerfile Requirements

- Must be ARM64 compatible (Raspberry Pi 5 cluster)
- Should use multi-stage builds for smaller images
- Must expose a single HTTP port
- Should include health check endpoint

Example Dockerfile:
```dockerfile
FROM docker.io/node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM docker.io/node:20-alpine
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY src ./src
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
CMD ["node", "src/index.js"]
```

### Application Requirements

- Must respond on port specified in Dockerfile
- Should provide `/health` or `/healthz` endpoint for Kubernetes probes
- Should log to stdout/stderr for proper log aggregation
- Should handle SIGTERM gracefully for clean shutdowns

## Related Repositories

- [homelab](https://github.com/kifbv/homelab) - Kubernetes infrastructure and application manifests

## CI/CD Architecture

```
┌─────────────────┐
│  User / n8n     │
│  pushes code    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ GitHub Webhook  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  n8n Workflow   │
│  - Parse commit │
│  - Get version  │
│  - Bump semver  │
│  - Create Job   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Kaniko Build   │
│  (in cluster)   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Registry     │
│ registry.k8s-   │
│   lab.dev       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Flux Image Auto │
│ - Detect new tag│
│ - Update YAML   │
│ - Commit & push │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   Flux Sync     │
│  Deploy to K8s  │
└─────────────────┘
```

## Troubleshooting

### Build Fails

Check Kaniko Job logs:
```bash
kubectl -n build-system get jobs
kubectl -n build-system logs job/<job-name>
```

### Image Not Found

Verify image was pushed to registry:
```bash
curl https://registry.k8s-lab.dev/v2/webapp/<app-name>/tags/list
```

### Flux Not Updating

Check ImageRepository and ImagePolicy:
```bash
kubectl -n apps get imagerepository <app-name>
kubectl -n apps get imagepolicy <app-name>
flux reconcile image repository <app-name>
```
