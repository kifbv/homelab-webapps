# n8n Workflow Documentation

This document describes the n8n automation workflows for building and deploying webapps to the Kubernetes cluster.

## Overview

We have two main workflows:

1. **New Webapp Creation** - Creates a new webapp from scratch with generated code
2. **Webapp Update Automation** - Automatically builds and deploys updates when code is pushed

## Architecture

```
┌─────────────────┐
│  User Request   │
│  (n8n trigger)  │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│            Workflow 1: New Webapp Creation              │
│                                                         │
│  1. Generate webapp code (AI or template)               │
│  2. Push code to homelab-webapps repo                   │
│  3. Build image with BuildKit                           │
│  4. Create Kubernetes manifests                         │
│  5. Push manifests to homelab repo                      │
│  6. Flux deploys the webapp                             │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│         Workflow 2: Webapp Update Automation            │
│                                                         │
│  1. GitHub webhook (push event)                         │
│  2. Parse conventional commit                           │
│  3. Query registry for current version                  │
│  4. Calculate new semver                                │
│  5. Build image with BuildKit (new tag)                 │
│  6. Flux Image Automation detects new image             │
│  7. Flux updates manifest and deploys                   │
└─────────────────────────────────────────────────────────┘
```

## Workflow 1: New Webapp Creation

### Trigger

Manual trigger from n8n UI or HTTP endpoint

### Input Parameters

- `webappName`: Name of the webapp (e.g., "app-example")
- `description`: Description of what the app does
- `port`: Port the app listens on (default: 3000)
- `language`: Programming language (nodejs, python, go, etc.)

### Steps

#### 1. Generate Webapp Code

**Node**: Code Generator (Function/AI)

```javascript
// Example: Use AI or template to generate webapp code
const webappCode = {
  dockerfile: `FROM docker.io/node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src ./src
EXPOSE ${port}
HEALTHCHECK --interval=30s --timeout=3s CMD wget --spider http://localhost:${port}/health || exit 1
USER node
CMD ["node", "src/index.js"]`,

  packageJson: {
    name: webappName,
    version: "1.0.0",
    description: description,
    main: "src/index.js",
    dependencies: {
      express: "^4.18.2"
    }
  },

  srcIndexJs: `const express = require('express');
const app = express();
const PORT = process.env.PORT || ${port};

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.get('/', (req, res) => {
  res.json({ message: '${description}' });
});

app.listen(PORT, () => console.log(\`Server running on port \${PORT}\`));`
};
```

#### 2. Push Code to homelab-webapps Repository

**Node**: Git Push (Execute Command)

```bash
# Clone homelab-webapps repo
git clone git@github.com-webapps:kifbv/homelab-webapps.git /tmp/webapps
cd /tmp/webapps

# Create app directory
mkdir -p apps/${webappName}
cd apps/${webappName}

# Write generated files
echo "${dockerfileContent}" > Dockerfile
echo "${packageJsonContent}" > package.json
mkdir -p src
echo "${srcIndexJsContent}" > src/index.js

# Generate package-lock.json
npm install

# Commit and push
cd /tmp/webapps
git add apps/${webappName}
git commit -m "feat(${webappName}): initial commit

- Add ${description}
- Port: ${port}
- Language: ${language}

🤖 Generated with n8n automation"
git push
```

#### 3. Build Image with BuildKit

**Node**: Create BuildKit Job (Kubernetes)

```javascript
// Read BuildKit Job template
const templatePath = '/mnt/homelab/kubernetes/rpi-cluster/infrastructure/build-system/buildkit-job-template.yaml';
const templateYaml = await fs.readFile(templatePath, 'utf8');

// Substitute placeholders
const jobName = `build-${webappName}-${Date.now()}`;
const imageTag = `registry.k8s-lab.dev/webapp/${webappName}:v1.0.0`;

const jobYaml = templateYaml
  .replace(/\${JOB_NAME}/g, jobName)
  .replace(/\${WEBAPP_NAME}/g, webappName)
  .replace(/\${GIT_REPO_URL}/g, 'https://github.com/kifbv/homelab-webapps.git')
  .replace(/\${GIT_SUBDIRECTORY}/g, `apps/${webappName}`)
  .replace(/\${IMAGE_TAG}/g, imageTag);

// Apply Job via kubectl
await exec(`echo '${jobYaml}' | kubectl apply -f -`);
```

#### 4. Wait for Build Completion

**Node**: Wait for Job (Kubernetes)

```javascript
// Poll for Job completion (max 5 minutes)
const maxWaitTime = 300000; // 5 minutes
const pollInterval = 5000; // 5 seconds
const startTime = Date.now();

while (Date.now() - startTime < maxWaitTime) {
  const jobStatus = await exec(`kubectl -n build-system get job ${jobName} -o jsonpath='{.status.succeeded}'`);

  if (jobStatus === '1') {
    console.log('Build completed successfully');
    break;
  }

  const failed = await exec(`kubectl -n build-system get job ${jobName} -o jsonpath='{.status.failed}'`);
  if (failed === '1') {
    const logs = await exec(`kubectl -n build-system logs job/${jobName}`);
    throw new Error(`Build failed:\n${logs}`);
  }

  await sleep(pollInterval);
}

// Get build logs for reference
const logs = await exec(`kubectl -n build-system logs job/${jobName}`);
```

#### 5. Create Kubernetes Manifests

**Node**: Generate Manifests (Function)

```javascript
const manifests = {
  namespace: `apiVersion: v1
kind: Namespace
metadata:
  name: ${webappName}`,

  deployment: `apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${webappName}
  namespace: ${webappName}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${webappName}
  template:
    metadata:
      labels:
        app: ${webappName}
    spec:
      containers:
      - name: ${webappName}
        image: registry.k8s-lab.dev/webapp/${webappName}:v1.0.0
        ports:
        - containerPort: ${port}
        livenessProbe:
          httpGet:
            path: /health
            port: ${port}
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: ${port}
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: 50m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi`,

  service: `apiVersion: v1
kind: Service
metadata:
  name: ${webappName}
  namespace: ${webappName}
spec:
  selector:
    app: ${webappName}
  ports:
  - port: 80
    targetPort: ${port}`,

  httpRoute: `apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ${webappName}
  namespace: ${webappName}
spec:
  parentRefs:
  - name: internal-gateway
    namespace: gateway
  hostnames:
  - "${webappName}.k8s-lab.dev"
  rules:
  - backendRefs:
    - name: ${webappName}
      port: 80`
};
```

#### 6. Push Manifests to homelab Repository

**Node**: Git Push Manifests (Execute Command)

```bash
# Clone homelab repo
git clone git@github.com-homelab:kifbv/homelab.git /tmp/homelab
cd /tmp/homelab/kubernetes/rpi-cluster/apps/${webappName}

# Create app directory structure
mkdir -p app

# Write manifests
echo "${namespaceYaml}" > app/namespace.yaml
echo "${deploymentYaml}" > app/deployment.yaml
echo "${serviceYaml}" > app/service.yaml
echo "${httpRouteYaml}" > app/httproute.yaml

# Create kustomization
cat > app/kustomization.yaml <<EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - deployment.yaml
  - service.yaml
  - httproute.yaml
EOF

# Create Flux Kustomization
cat > ks.yaml <<EOF
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: ${webappName}
  namespace: flux-system
spec:
  interval: 10m
  path: ./kubernetes/rpi-cluster/apps/${webappName}/app
  prune: true
  sourceRef:
    kind: GitRepository
    name: homelab
  targetNamespace: ${webappName}
EOF

# Update cluster apps kustomization
cd /tmp/homelab/kubernetes/rpi-cluster/apps
echo "  - ${webappName}/ks.yaml" >> kustomization.yaml

# Commit and push
cd /tmp/homelab
git add apps/${webappName}
git commit -m "feat(${webappName}): deploy webapp

- Add Kubernetes manifests
- Configure HTTPRoute for ${webappName}.k8s-lab.dev
- Initial version: v1.0.0

🤖 Generated with n8n automation"
git push
```

### Success Notification

The workflow should send a notification with:
- Webapp name
- URL: `https://${webappName}.k8s-lab.dev`
- Image tag: `v1.0.0`
- Build logs link

## Workflow 2: Webapp Update Automation

### Trigger

GitHub webhook for push events on `homelab-webapps` repository

### Webhook Configuration

**URL**: `https://n8n.k8s-lab.dev/webhook/webapp-update`

**Events**: `push`

**Secret**: Store in n8n credentials

### Steps

#### 1. Parse Webhook Payload

**Node**: Webhook (Trigger)

```javascript
const payload = $input.all()[0].json;
const branch = payload.ref.replace('refs/heads/', '');

// Only process main branch
if (branch !== 'main') {
  return;
}

// Extract commit information
const commits = payload.commits || [];
const latestCommit = commits[commits.length - 1];
const commitMessage = latestCommit.message;

// Extract changed files to determine which app was updated
const changedFiles = [
  ...latestCommit.added,
  ...latestCommit.modified,
  ...latestCommit.removed
];

// Determine webapp name from path (e.g., "apps/app-example/src/index.js" -> "app-example")
const webappPaths = changedFiles
  .filter(f => f.startsWith('apps/'))
  .map(f => f.split('/')[1]);

const webappName = [...new Set(webappPaths)][0]; // Get unique webapp name

if (!webappName) {
  console.log('No webapp changes detected');
  return;
}
```

#### 2. Parse Conventional Commit

**Node**: Parse Commit (Function)

```javascript
// Parse conventional commit format
const commitRegex = /^(feat|fix|docs|style|refactor|perf|test|chore|build|ci|revert)(\(.+\))?(!)?:\s*(.+)/;
const match = commitMessage.match(commitRegex);

if (!match) {
  console.log('Not a conventional commit, skipping build');
  return;
}

const commitType = match[1]; // feat, fix, etc.
const breakingChange = match[3] === '!';
const commitDescription = match[4];

// Check for BREAKING CHANGE in commit body
const hasBreakingInBody = commitMessage.includes('BREAKING CHANGE:');

// Determine version bump
let bumpType;
if (breakingChange || hasBreakingInBody) {
  bumpType = 'major'; // 1.0.0 -> 2.0.0
} else if (commitType === 'feat') {
  bumpType = 'minor'; // 1.0.0 -> 1.1.0
} else if (commitType === 'fix') {
  bumpType = 'patch'; // 1.0.0 -> 1.0.1
} else {
  console.log(`Commit type '${commitType}' does not trigger build`);
  return;
}
```

#### 3. Query Registry for Current Version

**Node**: Get Current Version (Kubernetes Exec)

```javascript
// Use registry-query pod to get current tags
const podName = await exec(`kubectl -n build-system get pods -l app=registry-query -o jsonpath='{.items[0].metadata.name}'`);

const tags = await exec(`kubectl -n build-system exec ${podName} -- crane ls docker-registry.registry.svc.cluster.local:5000/webapp/${webappName}`);

// Parse semver tags
const semverRegex = /^v?(\d+)\.(\d+)\.(\d+)$/;
const versions = tags.split('\n')
  .map(tag => {
    const match = tag.match(semverRegex);
    if (match) {
      return {
        tag: tag,
        major: parseInt(match[1]),
        minor: parseInt(match[2]),
        patch: parseInt(match[3])
      };
    }
  })
  .filter(v => v !== undefined);

// Find latest version
if (versions.length === 0) {
  throw new Error(`No versions found for ${webappName}`);
}

const latestVersion = versions.reduce((latest, current) => {
  if (current.major > latest.major) return current;
  if (current.major === latest.major && current.minor > latest.minor) return current;
  if (current.major === latest.major && current.minor === latest.minor && current.patch > latest.patch) return current;
  return latest;
});
```

#### 4. Calculate New Version

**Node**: Bump Version (Function)

```javascript
let newVersion;

if (bumpType === 'major') {
  newVersion = {
    major: latestVersion.major + 1,
    minor: 0,
    patch: 0
  };
} else if (bumpType === 'minor') {
  newVersion = {
    major: latestVersion.major,
    minor: latestVersion.minor + 1,
    patch: 0
  };
} else { // patch
  newVersion = {
    major: latestVersion.major,
    minor: latestVersion.minor,
    patch: latestVersion.patch + 1
  };
}

const newTag = `v${newVersion.major}.${newVersion.minor}.${newVersion.patch}`;
```

#### 5. Build Image with BuildKit

**Node**: Create BuildKit Job (Same as Workflow 1)

Use the same BuildKit Job creation as Workflow 1, but with the new version tag.

#### 6. Wait for Flux Image Automation

**Node**: Wait (Passive)

No action needed - Flux Image Automation will:

1. Detect new image tag via ImageRepository
2. Apply ImagePolicy to select latest version
3. Update manifest via ImageUpdateAutomation
4. Commit updated manifest to Git
5. Deploy updated application

### Success Notification

The workflow should send a notification with:
- Webapp name
- Old version → New version
- Commit message
- Build logs link

## Flux Image Automation Configuration

For each webapp, create these resources:

### ImageRepository

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: ${webappName}
  namespace: ${webappName}
spec:
  image: registry.k8s-lab.dev/webapp/${webappName}
  interval: 1m
  secretRef:
    name: registry-credentials
```

### ImagePolicy

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: ${webappName}
  namespace: ${webappName}
spec:
  imageRepositoryRef:
    name: ${webappName}
  policy:
    semver:
      range: '>=1.0.0'
```

### ImageUpdateAutomation

```yaml
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: ${webappName}
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: homelab
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@k8s-lab.dev
        name: Flux Image Automation
      messageTemplate: |
        chore(${webappName}): update image to {{range .Updated.Images}}{{println .}}{{end}}
    push:
      branch: main
  update:
    path: ./kubernetes/rpi-cluster/apps/${webappName}
    strategy: Setters
```

### Image Marker in Deployment

Add this comment to the deployment image field:

```yaml
spec:
  template:
    spec:
      containers:
      - name: ${webappName}
        image: registry.k8s-lab.dev/webapp/${webappName}:v1.0.0 # {"$imagepolicy": "${webappName}:${webappName}"}
```

## Helper Functions

### Create BuildKit Job

```javascript
async function createBuildKitJob(webappName, imageTag, gitSubdirectory) {
  const templatePath = '/mnt/homelab/kubernetes/rpi-cluster/infrastructure/build-system/buildkit-job-template.yaml';
  const templateYaml = await fs.readFile(templatePath, 'utf8');

  const jobName = `build-${webappName}-${Date.now()}`;

  const jobYaml = templateYaml
    .replace(/\${JOB_NAME}/g, jobName)
    .replace(/\${WEBAPP_NAME}/g, webappName)
    .replace(/\${GIT_REPO_URL}/g, 'https://github.com/kifbv/homelab-webapps.git')
    .replace(/\${GIT_SUBDIRECTORY}/g, gitSubdirectory)
    .replace(/\${IMAGE_TAG}/g, imageTag);

  await exec(`echo '${jobYaml}' | kubectl apply -f -`);

  return jobName;
}
```

### Wait for Job Completion

```javascript
async function waitForJobCompletion(jobName, namespace, timeoutMs) {
  const pollInterval = 5000;
  const startTime = Date.now();

  while (Date.now() - startTime < timeoutMs) {
    const succeeded = await exec(`kubectl -n ${namespace} get job ${jobName} -o jsonpath='{.status.succeeded}'`);

    if (succeeded === '1') {
      return true;
    }

    const failed = await exec(`kubectl -n ${namespace} get job ${jobName} -o jsonpath='{.status.failed}'`);
    if (failed === '1') {
      const logs = await exec(`kubectl -n ${namespace} logs job/${jobName}`);
      throw new Error(`Job failed:\n${logs}`);
    }

    await new Promise(resolve => setTimeout(resolve, pollInterval));
  }

  throw new Error(`Job ${jobName} timed out after ${timeoutMs}ms`);
}
```

### Get Latest Image Version

```javascript
async function getLatestVersion(webappName) {
  const podName = await exec(`kubectl -n build-system get pods -l app=registry-query -o jsonpath='{.items[0].metadata.name}'`);

  const tags = await exec(`kubectl -n build-system exec ${podName} -- crane ls docker-registry.registry.svc.cluster.local:5000/webapp/${webappName}`);

  const semverRegex = /^v?(\d+)\.(\d+)\.(\d+)$/;
  const versions = tags.split('\n')
    .map(tag => {
      const match = tag.match(semverRegex);
      if (match) {
        return {
          tag: tag,
          major: parseInt(match[1]),
          minor: parseInt(match[2]),
          patch: parseInt(match[3])
        };
      }
    })
    .filter(v => v !== undefined);

  if (versions.length === 0) {
    return { major: 0, minor: 0, patch: 0 };
  }

  return versions.reduce((latest, current) => {
    if (current.major > latest.major) return current;
    if (current.major === latest.major && current.minor > latest.minor) return current;
    if (current.major === latest.major && current.minor === latest.minor && current.patch > latest.patch) return current;
    return latest;
  });
}
```

### Bump Semver

```javascript
function bumpVersion(version, bumpType) {
  if (bumpType === 'major') {
    return {
      major: version.major + 1,
      minor: 0,
      patch: 0
    };
  } else if (bumpType === 'minor') {
    return {
      major: version.major,
      minor: version.minor + 1,
      patch: 0
    };
  } else { // patch
    return {
      major: version.major,
      minor: version.minor,
      patch: version.patch + 1
    };
  }
}

function versionToTag(version) {
  return `v${version.major}.${version.minor}.${version.patch}`;
}
```

## Testing

### Test Workflow 1: New Webapp Creation

1. Trigger workflow with test parameters
2. Verify code is pushed to homelab-webapps
3. Verify BuildKit Job runs successfully
4. Verify image is in registry
5. Verify manifests are pushed to homelab
6. Verify Flux deploys the webapp
7. Verify webapp is accessible at `https://${webappName}.k8s-lab.dev`

### Test Workflow 2: Webapp Update

1. Make a change to example app code
2. Commit with conventional commit: `fix(app-example): improve error handling`
3. Push to homelab-webapps
4. Verify webhook triggers workflow
5. Verify version is bumped (e.g., v1.0.0 → v1.0.1)
6. Verify BuildKit Job runs successfully
7. Verify new image is in registry
8. Verify Flux Image Automation updates manifest
9. Verify webapp is updated

## Security Considerations

1. **GitHub SSH Keys**: Use separate deploy keys for each repository
2. **Registry Credentials**: Stored as Kubernetes Secrets, encrypted with SOPS
3. **Webhook Secret**: Store in n8n credentials, verify signature
4. **RBAC**: n8n has minimal permissions (only build-system namespace)
5. **BuildKit Jobs**: Run in isolated namespace with privileged mode for builds only

## Troubleshooting

### Build fails

- Check BuildKit Job logs: `kubectl -n build-system logs job/<job-name>`
- Verify registry credentials exist
- Verify git repository is accessible

### Flux doesn't detect new image

- Check ImageRepository: `kubectl -n <webapp-name> get imagerepository`
- Check ImagePolicy: `kubectl -n <webapp-name> get imagepolicy`
- Verify registry credentials are reflected to webapp namespace

### Webhook not triggering

- Verify webhook configuration in GitHub
- Check n8n webhook URL is accessible
- Verify webhook secret matches
- Check n8n execution logs

### Version detection fails

- Verify registry-query pod is running: `kubectl -n build-system get pods -l app=registry-query`
- Test crane manually: `kubectl -n build-system exec deployment/registry-query -- crane ls <image>`
