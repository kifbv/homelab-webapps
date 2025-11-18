# Example Webapp Template

This is a template for creating new webapps in the homelab-webapps repository.

## Structure

```
_example/
├── Dockerfile          # Container image definition
├── package.json        # Node.js dependencies
├── src/
│   └── index.js        # Application entry point
└── README.md           # This file
```

## Features

- **Express.js** web server
- **Health check** endpoint at `/health`
- **Readiness** probe at `/ready`
- **Graceful shutdown** on SIGTERM
- **Multi-stage** Docker build
- **Non-root** user in container
- **ARM64** compatible (Raspberry Pi 5)

## Endpoints

- `GET /` - Root endpoint with API information
- `GET /health` - Health check (for Kubernetes liveness probe)
- `GET /ready` - Readiness check (for Kubernetes readiness probe)
- `GET /api/hello?name=World` - Example API endpoint

## Development

### Local Testing

```bash
npm install
npm start
```

### Build Container

```bash
docker build -t example-webapp:latest .
docker run -p 3000:3000 example-webapp:latest
```

### Test Health Check

```bash
curl http://localhost:3000/health
curl http://localhost:3000/api/hello?name=Homelab
```

## Deployment

This webapp is designed to be deployed via the n8n workflow automation system:

1. Code is committed to `app-*` directory
2. GitHub webhook triggers n8n
3. n8n builds image with Kaniko
4. Image is pushed to registry.k8s-lab.dev
5. n8n creates Kubernetes manifests in homelab repo
6. Flux deploys to cluster

## Kubernetes Manifest Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-webapp
  namespace: apps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-webapp
  template:
    metadata:
      labels:
        app: example-webapp
    spec:
      containers:
      - name: webapp
        image: registry.k8s-lab.dev/webapp/example-webapp:v1.0.0 # {"$imagepolicy": "apps:example-webapp"}
        ports:
        - containerPort: 3000
          name: http
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: example-webapp
  namespace: apps
spec:
  selector:
    app: example-webapp
  ports:
  - port: 80
    targetPort: 3000
    name: http
```

## Customization

To create a new webapp from this template:

1. Copy `_example/` to `app-myapp/`
2. Modify `package.json` (name, description, dependencies)
3. Implement your application logic in `src/`
4. Update `Dockerfile` if needed (different base image, build steps)
5. Commit with conventional commit message
6. Push to repository

Example:
```bash
cp -r _example app-myapp
cd app-myapp
# Make your changes
git add app-myapp
git commit -m "feat: add myapp application"
git push
```
