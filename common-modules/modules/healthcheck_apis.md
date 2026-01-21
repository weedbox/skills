# healthcheck_apis

Kubernetes-compatible health check endpoints.

## Installation

```bash
go get github.com/weedbox/common-modules/healthcheck_apis
```

## Usage in Fx

```go
import "github.com/weedbox/common-modules/healthcheck_apis"

func loadModules() ([]fx.Option, error) {
    return []fx.Option{
        http_server.Module("http_server"),
        healthcheck_apis.Module("healthcheck"),
        daemon.Module("daemon"),  // Required for status tracking
    }, nil
}
```

## Dependencies

This module requires:
- `http_server` - For endpoint registration
- `daemon` - For health/readiness status

## Endpoints

| Endpoint | Purpose | Success | Failure |
|----------|---------|---------|---------|
| `GET /healthz` | Liveness probe | 200 `{"status": "ok"}` | 500 `{"status": "unhealthy"}` |
| `GET /ready` | Readiness probe | 200 `{"ready": true}` | 500 `{"ready": false}` |

## Response Examples

### Healthy Service

```bash
$ curl http://localhost:8080/healthz
{"status": "ok"}

$ curl http://localhost:8080/ready
{"ready": true}
```

### Unhealthy/Not Ready Service

```bash
$ curl http://localhost:8080/healthz
{"status": "unhealthy"}

$ curl http://localhost:8080/ready
{"ready": false}
```

## Kubernetes Configuration

### Pod Specification

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: myapp:latest
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
```

### Deployment Specification

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

## How It Works

1. **Liveness (`/healthz`)**:
   - Returns the daemon's health status
   - Kubernetes restarts the pod if this fails repeatedly
   - Use for detecting deadlocks or unrecoverable states

2. **Readiness (`/ready`)**:
   - Returns true only after all modules are initialized
   - Kubernetes removes pod from service load balancer if not ready
   - Use for graceful startup/shutdown

## Complete Example

```go
func initModules() ([]fx.Option, error) {
    return []fx.Option{
        fx.Supply(config),
        logger.Module(),

        // HTTP and health checks
        http_server.Module("http_server"),
        healthcheck_apis.Module("healthcheck"),

        // Your application modules
        mymodule.Module("mymodule"),

        // Daemon must be last (marks service ready)
        daemon.Module("daemon"),
    }, nil
}
```
