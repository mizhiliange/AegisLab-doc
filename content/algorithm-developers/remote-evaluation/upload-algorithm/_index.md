---
title: Upload Algorithm
weight: 1
---

Push your containerized algorithm to the Harbor registry for remote execution.

## Prerequisites

- Docker installed and running
- Algorithm containerized (see [Containerization](../../development-guide/containerization))
- Harbor registry credentials
- Algorithm tested locally

## Step 1: Build Docker Image

Build your algorithm image with proper tagging:

```bash
cd my-rca-algorithm
docker build -t my-rca:latest .
```

Verify the image builds successfully:

```bash
docker images | grep my-rca
```

## Step 2: Tag for Harbor Registry

Tag your image for the Harbor registry:

```bash
# Format: harbor.example.com/project/algorithm:version
docker tag my-rca:latest harbor.aegislab.io/algorithms/my-rca:v1.0.0
```

Use semantic versioning for tags:
- `v1.0.0`: Major release
- `v1.1.0`: Minor update (new features)
- `v1.0.1`: Patch (bug fixes)
- `latest`: Current stable version

## Step 3: Login to Harbor

Authenticate with the Harbor registry:

```bash
docker login harbor.aegislab.io
```

Enter your credentials when prompted:
- Username: Your Harbor username
- Password: Your Harbor password or CLI secret

## Step 4: Push Image

Push the tagged image to Harbor:

```bash
docker push harbor.aegislab.io/algorithms/my-rca:v1.0.0
```

Push the latest tag as well:

```bash
docker tag my-rca:latest harbor.aegislab.io/algorithms/my-rca:latest
docker push harbor.aegislab.io/algorithms/my-rca:latest
```

## Step 5: Verify Upload

Check the image is available in Harbor:

```bash
# Using Harbor CLI
harbor list-images algorithms/my-rca

# Or via web UI
# Navigate to: https://harbor.aegislab.io/harbor/projects/1/repositories
```

## Using the CLI Helper

The rcabench-platform provides a helper command:

```bash
cd rcabench-platform
./main.py upload-algorithm-harbor my-rca v1.0.0
```

This command:
1. Builds the Docker image
2. Tags it appropriately
3. Pushes to Harbor registry
4. Verifies the upload

## Image Requirements

Your Docker image must:

1. **Include entrypoint.sh**: Entry point script that handles algorithm execution
2. **Include info.toml**: Metadata file with algorithm information
3. **Accept standard arguments**: `--trace-path`, `--ground-truth-path`, `--output-path`, etc.
4. **Write results to output path**: JSON file with ranked services
5. **Use base image**: Recommended to use `python:3.10-slim` or similar

Example Dockerfile structure:

```dockerfile
FROM python:3.10-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy algorithm code
COPY src/ ./src/
COPY entrypoint.sh info.toml ./

# Make entrypoint executable
RUN chmod +x entrypoint.sh

ENTRYPOINT ["./entrypoint.sh"]
```

## Image Size Optimization

Reduce image size for faster deployment:

```dockerfile
# Use slim base image
FROM python:3.10-slim

# Multi-stage build for compiled dependencies
FROM python:3.10 as builder
RUN pip install --user polars pandas numpy
FROM python:3.10-slim
COPY --from=builder /root/.local /root/.local

# Clean up
RUN apt-get clean && rm -rf /var/lib/apt/lists/*
```

## Security Best Practices

1. **Don't include secrets**: Never hardcode API keys or credentials
2. **Use specific versions**: Pin dependency versions in requirements.txt
3. **Scan for vulnerabilities**: Use `docker scan` before pushing
4. **Minimal base image**: Use slim or alpine variants
5. **Non-root user**: Run as non-root user when possible

```dockerfile
# Create non-root user
RUN useradd -m -u 1000 algorithm
USER algorithm
```

## Troubleshooting

### Authentication Failed

```
Error: unauthorized: authentication required
```

**Solution**: Login to Harbor registry:

```bash
docker login harbor.aegislab.io
```

### Push Denied

```
Error: denied: requested access to the resource is denied
```

**Solution**: Verify you have push permissions to the `algorithms` project. Contact your administrator.

### Image Too Large

```
Warning: Image size is 2.5GB
```

**Solution**: Optimize your image:
- Use slim base images
- Remove build dependencies
- Use multi-stage builds
- Clean up package manager caches

### Build Fails

```
Error: failed to solve: process "/bin/sh -c pip install -r requirements.txt" did not complete successfully
```

**Solution**: Check requirements.txt for invalid packages or version conflicts. Test locally first.

## Next Steps

- [Submit Execution](../submit-execution): Create evaluation tasks
- [Monitor Progress](../monitor-progress): Track execution status
