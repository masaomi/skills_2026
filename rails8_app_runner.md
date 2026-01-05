# Rails 8 + AWS App Runner Deployment Tips

## Overview

Rails 8 uses **Thruster** as a reverse proxy in front of Puma. When deploying to AWS App Runner, there are specific configurations required due to:
- Non-root user restrictions (cannot bind to ports below 1024)
- Thruster's environment variable naming
- Architecture requirements (linux/amd64)

---

## Key Points

### 1. Port Configuration (CRITICAL)

**Problem**: Rails 8's Thruster defaults to port 80, but non-root users cannot bind to ports below 1024.

**Error you'll see**:
```json
{"level":"ERROR","msg":"Failed to start HTTP listener","error":"listen tcp :80: bind: permission denied"}
```

**Solution**: Use port 8080 instead.

| Setting | Value |
|---------|-------|
| App Runner Port | `8080` |
| Environment Variable | `HTTP_PORT=8080` |

**Important**: Thruster uses `HTTP_PORT`, NOT `PORT`.

### 2. Thruster Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `HTTP_PORT` | Port Thruster listens on | 80 |
| `TARGET_PORT` | Port Puma listens on (internal) | 3000 |

### 3. Dockerfile Configuration

```dockerfile
# Key parts for App Runner compatibility
FROM ruby:3.3.10-slim AS base

# ... build stages ...

# Final stage - run as non-root user
RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash

# Create necessary directories BEFORE switching to non-root user
RUN mkdir -p /rails/storage /rails/db /rails/log /rails/tmp && \
    chown -R rails:rails /rails/storage /rails/db /rails/log /rails/tmp

USER 1000:1000

ENTRYPOINT ["/rails/bin/docker-entrypoint"]

# Use port 8080, not 80
EXPOSE 8080
ENV HTTP_PORT=8080
ENV TARGET_PORT=3000
CMD ["./bin/thrust", "./bin/rails", "server", "-p", "3000"]
```

### 4. Docker Build Command

**Always specify `--platform linux/amd64`** (especially on Apple Silicon Macs):

```bash
docker build --platform linux/amd64 -f docker/Dockerfile.backend -t myapp-backend backend/
```

### 5. Required Environment Variables in App Runner

| Name | Value | Notes |
|------|-------|-------|
| `RAILS_ENV` | `production` | Required |
| `RAILS_LOG_TO_STDOUT` | `1` | For CloudWatch logging |
| `RAILS_MASTER_KEY` | `<content of config/master.key>` | Required for credentials |
| `SECRET_KEY_BASE` | `<generate with bin/rails secret>` | Required |
| `HTTP_PORT` | `8080` | **Thruster listen port** |

### 6. App Runner Service Settings

| Setting | Recommended Value |
|---------|-------------------|
| Port | `8080` |
| Virtual CPU | 0.5 vCPU (minimum for Rails) |
| Virtual memory | 1 GB (minimum for Rails) |
| Health check path | `/health` |
| Health check timeout | 10 seconds |

### 7. Health Check Endpoint

Add a simple health endpoint to your Rails app:

```ruby
# config/routes.rb
get '/health', to: 'health#index'

# app/controllers/health_controller.rb
class HealthController < ApplicationController
  def index
    render json: { status: 'ok', rails_version: Rails.version }
  end
end
```

---

## Troubleshooting

### Error: `listen tcp :80: bind: permission denied`
- **Cause**: Non-root user cannot bind to port 80
- **Solution**: Set `HTTP_PORT=8080` in environment variables AND App Runner port to 8080

### Error: `Container exit code: 1` or `255`
- Check CloudWatch Logs → Log groups → `/aws/apprunner/<service-name>/application`
- Common causes:
  - Missing `RAILS_MASTER_KEY`
  - Wrong port configuration
  - Insufficient memory (use at least 1GB)
  - Architecture mismatch (need `linux/amd64`)

### Build cache issues
```bash
# Force rebuild without cache
docker builder prune -af
docker build --no-cache --platform linux/amd64 -f docker/Dockerfile.backend -t myapp backend/
```

### Image not updating in App Runner
After pushing to ECR, click "Deploy" in App Runner console to pull the latest image.

---

## Architecture Diagram

```
Internet
    ↓
App Runner (Port 8080)
    ↓
Thruster (HTTP_PORT=8080) ← Handles compression, caching
    ↓
Puma (TARGET_PORT=3000)   ← Rails application
```

---

## ECR Push Commands

```bash
# Set variables
export AWS_ACCOUNT_ID=<your-account-id>
export REGION=<your-region>

# Login to ECR
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

# Build for linux/amd64
docker build --platform linux/amd64 -f docker/Dockerfile.backend -t myapp-backend backend/

# Tag and push
docker tag myapp-backend:latest $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/myapp-backend:latest
docker push $AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/myapp-backend:latest
```

---

## References

- [Rails 8 Release Notes](https://guides.rubyonrails.org/8_0_release_notes.html)
- [Thruster GitHub](https://github.com/basecamp/thruster)
- [AWS App Runner Documentation](https://docs.aws.amazon.com/apprunner/)


