# Example Dockerfile for Node.js webapp
# Multi-stage build for smaller final image

FROM docker.io/node:20-alpine AS builder
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Final stage
FROM docker.io/node:20-alpine
WORKDIR /app

# Copy dependencies from builder
COPY --from=builder /app/node_modules ./node_modules

# Copy application code
COPY src ./src

# Expose application port
EXPOSE 3000

# Health check for Kubernetes probes
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Run as non-root user
USER node

# Start application
CMD ["node", "src/index.js"]
