# DevTide Agent - Claude.md

## Project Overview
The DevTide Agent is a cluster-side component of the DevTide platform that runs within each managed Kubernetes cluster. It serves as the execution layer for infrastructure operations, handling Terraform/OpenTofu executions, drift detection, and status reporting back to the DevTide Operator.

## Architecture Overview
The agent operates as a lightweight daemon within the cluster, providing:
1. **Job Execution**: Running Terraform/OpenTofu operations in isolated containers
2. **Drift Detection**: Monitoring infrastructure state for configuration drift
3. **Status Reporting**: Streaming real-time status updates to the operator
4. **Resource Management**: Managing job resources and cleanup
5. **Security**: Secure credential handling and job isolation

## Key Responsibilities

### Terraform/OpenTofu Execution
- Receives job specifications from the operator
- Spawns isolated Kubernetes Jobs for Terraform runs
- Manages state backend configuration
- Handles provider credentials securely
- Streams logs and status back to operator

### Drift Detection
- Periodically runs `terraform plan` to detect drift
- Reports drift status to operator
- Triggers notifications for drift events
- Configurable detection intervals per deployment

### Resource Monitoring
- Tracks job resource usage (CPU, memory)
- Implements resource limits and quotas
- Cleans up completed job resources
- Reports metrics to observability stack

## Communication Design

### Agent to Operator Communication
Based on the AGENT_OPERATOR_COMMUNICATION_DESIGN.md:

```
┌─────────────────┐         gRPC          ┌──────────────┐
│   DevTide       │◄──────────────────────►│   DevTide    │
│   Operator      │    Bidirectional       │   Agent      │
│                 │      Streaming         │              │
└─────────────────┘                        └──────────────┘
        │                                          │
        │                                          │
        ▼                                          ▼
┌─────────────────┐                        ┌──────────────┐
│   Frontend      │                        │  Kubernetes  │
│   (Status Agg)  │                        │    Jobs      │
└─────────────────┘                        └──────────────┘
```

### gRPC Service Definition
```protobuf
service AgentService {
  // Bidirectional streaming for job execution
  rpc ExecuteJob(stream JobRequest) returns (stream JobStatus);
  
  // Stream status updates to operator
  rpc StreamStatus(stream StatusUpdate) returns (stream OperatorCommand);
  
  // Health check
  rpc HealthCheck(HealthRequest) returns (HealthResponse);
}
```

## Repository Structure
```
devtide-agent/
├── cmd/agent/            # Agent main entry point
├── pkg/
│   ├── agent/           # Core agent logic
│   ├── executor/        # Job execution engine
│   ├── drift/           # Drift detection
│   ├── grpc/            # gRPC client/server
│   ├── job/             # Kubernetes job management
│   └── security/        # Credential handling
├── k8s/                  # Kubernetes manifests
│   ├── daemonset.yaml   # Agent DaemonSet
│   ├── rbac.yaml        # RBAC permissions
│   └── configmap.yaml   # Agent configuration
├── helm/
│   └── devtide-agent/   # Agent Helm chart
└── docs/                 # Documentation
```

## Development Workflow

### Prerequisites
- Go 1.20+
- Docker
- Kubernetes cluster access
- kubectl
- Make

### Local Development
```bash
# Clone the repository
git clone https://github.com/axyzlabs/devtide-agent.git
cd devtide-agent

# Install dependencies
go mod download

# Run locally (outside cluster)
make run-local

# Run tests
make test
```

### Building
```bash
# Build binary
make build

# Build Docker image
make docker-build

# Push Docker image
make docker-push
```

### Deployment
```bash
# Deploy to cluster
kubectl apply -f k8s/

# Or using Helm
helm install devtide-agent helm/devtide-agent/
```

## Configuration

### Environment Variables
```yaml
# Agent configuration
AGENT_ID: "cluster-1-agent"
OPERATOR_ENDPOINT: "devtide-operator.devtide-system:8080"
LOG_LEVEL: "info"
METRICS_PORT: "8080"

# Job execution
JOB_NAMESPACE: "devtide-jobs"
JOB_SERVICE_ACCOUNT: "devtide-job-runner"
JOB_IMAGE: "axyzlabs/terraform-runner:latest"
MAX_CONCURRENT_JOBS: "5"

# Security
TLS_ENABLED: "true"
TLS_CERT_PATH: "/etc/devtide/tls/cert.pem"
TLS_KEY_PATH: "/etc/devtide/tls/key.pem"
```

## Security Considerations

### RBAC Requirements
The agent requires specific Kubernetes permissions:
```yaml
- apiGroups: ["batch"]
  resources: ["jobs"]
  verbs: ["create", "get", "list", "watch", "delete"]
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list", "create", "update"]
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
```

### Credential Management
- Uses Kubernetes secrets for provider credentials
- Implements least-privilege access
- Rotates temporary credentials when possible
- Never logs sensitive information

## Job Execution Flow

1. **Job Request**: Operator sends job specification
2. **Validation**: Agent validates job parameters
3. **Resource Creation**: Creates ConfigMap with Terraform config
4. **Job Spawn**: Creates Kubernetes Job with appropriate image
5. **Monitoring**: Watches job progress and streams logs
6. **Status Updates**: Sends real-time updates to operator
7. **Cleanup**: Removes completed job resources

## Observability

### Metrics (Prometheus)
- `devtide_agent_jobs_total`: Total jobs executed
- `devtide_agent_jobs_active`: Currently running jobs
- `devtide_agent_jobs_failed`: Failed job count
- `devtide_agent_drift_detected`: Drift detection events
- `devtide_agent_grpc_connections`: Active gRPC connections

### Logging
- Structured JSON logging
- Log levels: debug, info, warn, error
- Correlation IDs for request tracing
- Integration with Loki for aggregation

### Tracing
- OpenTelemetry integration
- Distributed tracing for job execution
- gRPC interceptors for automatic tracing

## Testing Strategy

### Unit Tests
- Test coverage > 80%
- Mock Kubernetes client for job operations
- Test gRPC communication logic
- Validate error handling

### Integration Tests
- Test against real Kubernetes API
- Validate job execution flow
- Test operator communication
- Verify resource cleanup

### E2E Tests
- Full job execution scenarios
- Multi-cluster deployment tests
- Failure and recovery testing
- Performance benchmarks

## Contributing Guidelines

### Code Style
- Follow standard Go conventions
- Use golangci-lint
- Write comprehensive tests
- Document public APIs

### Pull Request Process
1. Fork and create feature branch
2. Write tests for new features
3. Update documentation
4. Submit PR with description
5. Ensure CI passes

## Related Repositories
- [devtide](https://github.com/axyzlabs/devtide): Main operator
- [devtide-frontend](https://github.com/axyzlabs/devtide-frontend): Web UI

## License
[License details to be determined]

---

*Part of the DevTide platform - GitOps-powered infrastructure management.*
