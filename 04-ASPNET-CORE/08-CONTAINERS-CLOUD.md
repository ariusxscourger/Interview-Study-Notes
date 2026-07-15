# CONTAINERS, CLOUD & DEPLOYMENT
## Exam-Style Study Notes

---

## TOPIC: Docker, Kubernetes & Cloud Deployment for ASP.NET Core

### WHY? (Problem Statement)

**Before Containers:**
- "Works on my machine" - environment differences
- Manual server provisioning, configuration drift
- Difficult scaling, no resource isolation
- Deployment = copy files, pray, rollback = restore backup

**What Containers Solve:**
| Problem | Container Solution |
|---------|-------------------|
| Environment consistency | Immutable image runs same everywhere |
| Dependency conflicts | Isolated filesystem, own dependencies |
| Scaling | Horizontal pod scaling in seconds |
| Resource efficiency | Lightweight vs VMs (no guest OS) |
| Deployment | Image = artifact, deploy same image to dev/stage/prod |
| Rollback | Previous image tag = instant rollback |

**Real-World Analogy:**
- VMs = Shipping entire houses (with furniture, plumbing, foundation)
- Containers = Shipping standardized containers (just the cargo)

---

### HOW? (Internal Mechanism)

#### 1. **Docker Image Layers (Union File System)**

```
Dockerfile:
FROM mcr.microsoft.com/dotnet/aspnet:8.0          ← Base Layer (Read-only)
WORKDIR /app                                      ← Metadata
COPY --from=build /app/publish .                  ← App Layer (Read-only)
ENTRYPOINT ["dotnet", "MyApp.dll"]                ← Metadata

Runtime:
┌─────────────────────────────────────┐
│  Container Layer (Read/Write)       │  ← Logs, temp files, runtime changes
├─────────────────────────────────────┤
│  App Layer (COPY)                   │  ← Your code
├─────────────────────────────────────┤
│  Runtime Layer (aspnet:8.0)         │  ← .NET runtime, libraries
├─────────────────────────────────────┤
│  Base OS Layer (debian/ubuntu)      │  ← Kernel shared with host
└─────────────────────────────────────┘
```

#### 2. **Multi-Stage Build (Production Optimization)**

```dockerfile
# Stage 1: Build (SDK image - larger, has compiler)
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app/publish --no-restore

# Stage 2: Runtime (ASP.NET image - smaller, no compiler)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
EXPOSE 8080
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Size Comparison:**
- SDK image: ~800MB
- ASP.NET Runtime: ~200MB
- Distroless/Alpine: ~100MB
- **Multi-stage: Only runtime + app in final image**

#### 3. **Kubernetes Architecture**

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL PLANE                            │
│  ┌─────────┐  ┌─────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │API Server│──│Scheduler│  │Controller Mgr│  │   etcd       │  │
│  └─────────┘  └─────────┘  └─────────────┘  └──────────────┘  │
└────────────────────────────┬────────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
         ┌─────────┐    ┌─────────┐    ┌─────────┐
         │ Node 1  │    │ Node 2  │    │ Node 3  │
         │         │    │         │    │         │
         │ ┌─────┐ │    │ ┌─────┐ │    │ ┌─────┐ │
         │ │Pod 1│ │    │ │Pod 3│ │    │ │Pod 5│ │
         │ │Pod 2│ │    │ │Pod 4│ │    │ │Pod 6│ │
         │ └─────┘ │    │ └─────┘ │    │ └─────┘ │
         │ kubelet │    │ kubelet │    │ kubelet │
         │ k-proxy │    │ k-proxy │    │ k-proxy │
         │Container│    │Container│    │Container│
         │ Runtime │    │ Runtime │    │ Runtime │
         └─────────┘    └─────────┘    └─────────┘
```

---

### WHAT? (Key Concepts & APIs)

#### 1. **Production Dockerfile for ASP.NET Core**

```dockerfile
# ---- Build Stage ----
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src

# Cache dependencies
COPY ["src/MyApp/MyApp.csproj", "src/MyApp/"]
COPY ["src/MyApp.Tests/MyApp.Tests.csproj", "src/MyApp.Tests/"]
RUN dotnet restore "src/MyApp/MyApp.csproj"

# Copy source and build
COPY . .
WORKDIR "/src/src/MyApp"
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish \
    --no-restore \
    -p:PublishReadyToRun=true \
    -p:PublishTrimmed=true \
    -p:TrimMode=partial

# ---- Runtime Stage ----
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime

# Non-root user for security
RUN addgroup -g 1000 -S appgroup && \
    adduser -u 1000 -S appuser -G appgroup

WORKDIR /app
EXPOSE 8080

# Copy published app
COPY --from=build --chown=appuser:appgroup /app/publish .

# Security: Read-only root filesystem (mostly)
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Build & Run:**
```bash
# Build
docker build -t myapp:1.0.0 -f src/MyApp/Dockerfile .

# Run locally
docker run -d -p 8080:8080 \
  -e ASPNETCORE_ENVIRONMENT=Production \
  -e ConnectionStrings__Default="Server=host.docker.internal;Database=MyDb;..." \
  myapp:1.0.0
```

#### 2. **Kubernetes Manifests**

**Deployment:**
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: myapp
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      containers:
      - name: myapp
        image: myregistry.azurecr.io/myapp:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__Default
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: connection-string
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

**Service:**
```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: myapp
```

**Ingress:**
```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 80
```

**Horizontal Pod Autoscaler:**
```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

#### 3. **Helm Chart Structure**

```
myapp/
├── Chart.yaml
├── values.yaml
├── values-prod.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── servicemonitor.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
└── charts/
```

**values.yaml:**
```yaml
replicaCount: 3

image:
  repository: myregistry.azurecr.io/myapp
  pullPolicy: IfNotPresent
  tag: "1.0.0"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.myapp.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - api.myapp.com

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 50
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

healthChecks:
  livenessPath: /health/live
  readinessPath: /health/ready
```

---

### PATTERNS & ANTI-PATTERNS

| Pattern | When to Use | Anti-Pattern | Why Avoid |
|---------|-------------|--------------|-----------|
| **Multi-stage Build** | Always for production | Single-stage with SDK | Image 4x larger, contains compiler |
| **Non-root User** | Always | Running as root | Container escape = root on host |
| **Health Checks** | Always | No probes | K8s can't detect unhealthy pods |
| **Resource Limits** | Always | No limits | OOM kills, noisy neighbors |
| **Rolling Update** | Stateless apps | Recreate strategy | Downtime on deploy |
| **Blue/Green or Canary** | Critical services | Direct production deploy | No rollback safety |
| **Distroless/Alpine** | Size/security critical | Full Debian base | Larger attack surface |
| **Secret Management** | All secrets | Env vars in git | Leaked credentials |

---

### INTERVIEW QUESTIONS (Top 15)

#### 1. **Conceptual**: "Explain the difference between a Docker Image and Container"

**Answer:**
```
Image: Read-only template with layers (filesystem + metadata). Immutable.
Container: Runnable instance of an image. Adds thin read-write layer on top.
            Has its own PID namespace, network, filesystem.
            
Analogy: Image = Class, Container = Object instance
         Image = VM template, Container = Running VM
```

#### 2. **Code**: "Write a Dockerfile that produces a <100MB image for ASP.NET Core"

```dockerfile
# Use distroless (no shell, no package manager)
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app -p:PublishTrimmed=true -p:TrimMode=partial

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine AS runtime
WORKDIR /app
COPY --from=build /app .
USER 1000
ENTRYPOINT ["dotnet", "MyApp.dll"]

# Result: ~80MB vs ~200MB standard
```

#### 3. **Design**: "How to handle database migrations in Kubernetes?"

```yaml
# Option 1: Init Container (runs before app)
initContainers:
- name: migrate
  image: myregistry.azurecr.io/myapp:1.0.0
  command: ["dotnet", "MyApp.dll", "--migrate"]
  envFrom:
  - secretRef:
      name: myapp-secrets

# Option 2: Job (separate, can retry)
apiVersion: batch/v1
kind: Job
metadata:
  name: myapp-migrate-{{ .Chart.AppVersion }}
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migrate
        image: myregistry.azurecr.io/myapp:1.0.0
        command: ["dotnet", "MyApp.dll", "--migrate"]
        envFrom:
        - secretRef:
            name: myapp-secrets

# Option 3: External migration tool (Flyway, DbUp) - Recommended for production
# Run in CI/CD pipeline BEFORE deploying new version
```

#### 4. **Performance**: "Container starts slow - how to optimize?"

```dockerfile
# 1. ReadyToRun (AOT-like)
RUN dotnet publish -p:PublishReadyToRun=true

# 2. Trimming (remove unused IL)
RUN dotnet publish -p:PublishTrimmed=true -p:TrimMode=partial

# 3. Native AOT (smallest, fastest startup - .NET 8+)
RUN dotnet publish -p:PublishAot=true -r linux-musl-x64

# 4. Layer caching - copy csproj first
COPY *.csproj .
RUN dotnet restore
COPY . .

# 5. Smaller base image
FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine  # Not full debian

# 6. Multi-arch for ARM (Apple Silicon, Graviton)
docker buildx build --platform linux/amd64,linux/arm64 -t myapp .
```

#### 5. **Code**: "Implement graceful shutdown in ASP.NET Core for K8s"

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// Increase shutdown timeout (default 30s)
builder.WebHost.ConfigureKestrel(options =>
{
    options.Limits.KeepAliveTimeout = TimeSpan.FromMinutes(2);
});

var app = builder.Build();

// Handle SIGTERM (K8s sends this before killing)
var lifetime = app.Services.GetRequiredService<IHostApplicationLifetime>();
var logger = app.Services.GetRequiredService<ILogger<Program>>();

lifetime.ApplicationStopping.Register(() =>
{
    logger.LogInformation("SIGTERM received, stopping gracefully...");
    // Stop accepting new requests (readiness probe will fail)
    // Finish processing current requests
    // Flush logs, close connections
});

lifetime.ApplicationStopped.Register(() =>
{
    logger.LogInformation("Application stopped");
});

app.Run();

// K8s config for graceful shutdown
// terminationGracePeriodSeconds: 60  (in deployment spec)
// preStop hook: sleep 10 (lets endpoints update)
```

```yaml
# In deployment.yaml
spec:
  terminationGracePeriodSeconds: 60
  template:
    spec:
      containers:
      - name: myapp
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
```

#### 6. **Architecture**: "Blue-Green vs Canary vs Rolling deployment"

| Strategy | How it Works | Pros | Cons |
|----------|--------------|------|------|
| **Rolling** (Default K8s) | Replace pods one by one | Zero-downtime, simple | No instant rollback, mixed versions |
| **Blue-Green** | Two identical envs, switch traffic | Instant rollback, full test | 2x resources, stateful issues |
| **Canary** | Route % traffic to new version | Gradual rollout, automated rollback | Complex, needs service mesh |

```yaml
# Canary with Argo Rollouts
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}
      - setWeight: 30
      - pause: {duration: 10m}
      - setWeight: 50
      - pause: {duration: 10m}
      - setWeight: 100
      canaryMetadata:
        labels:
          role: canary
      stableMetadata:
        labels:
          role: stable
      trafficRouting:
        nginx:
          stableIngress: myapp-stable
```

#### 7. **Security**: "Container security best practices"

```dockerfile
# 1. Non-root user
RUN addgroup -g 1000 -S appgroup && adduser -u 1000 -S appuser -G appgroup
USER appuser

# 2. Read-only root filesystem
# In K8s:
securityContext:
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  runAsNonRoot: true
  runAsUser: 1000

# 3. Scan images
# Trivy: trivy image myapp:latest
# In CI: fail on HIGH/CRITICAL

# 4. Minimal base (distroless)
FROM gcr.io/distroless/aspnet:8.0
# No shell, no package manager, no CVE surface

# 5. Sign images (cosign)
cosign sign --key env://COSIGN_PRIVATE_KEY myregistry.io/myapp:v1.0.0

# 6. Admission control (OPA Gatekeeper / Kyverno)
# Reject: root, privileged, latest tag, no resources
```

#### 8. **Debugging**: "Pod is CrashLoopBackOff - how to debug?"

```bash
# 1. Check events
kubectl describe pod myapp-xxx

# 2. Previous container logs (if restarted)
kubectl logs myapp-xxx --previous

# 3. Exec into running container (if briefly up)
kubectl exec -it myapp-xxx -- /bin/sh

# 4. Debug with ephemeral container (K8s 1.23+)
kubectl debug -it myapp-xxx --image=busybox --target=myapp

# 5. Check readiness/liveness endpoints
kubectl exec myapp-xxx -- wget -qO- http://localhost:8080/health/ready

# 6. Common causes:
# - Missing env vars / secrets
# - DB connection string wrong
# - Port already in use
# - OOMKilled (check: kubectl describe | grep OOM)
# - Permission denied (non-root user can't write)
```

#### 9. **Code**: "Health checks for Kubernetes"

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddDbContextCheck<AppDbContext>()  // DB connection
    .AddRedis(builder.Configuration.GetConnectionString("Redis"))  // Cache
    .AddUrlGroup(new Uri("https://api.external.com/health"), "external-api")  // External
    .AddCheck<CustomHealthCheck>("business-logic");

// Custom check
public class CustomHealthCheck : IHealthCheck
{
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, CancellationToken ct)
    {
        var isHealthy = CheckBusinessRules();
        return isHealthy 
            ? Task.FromResult(HealthCheckResult.Healthy("All good"))
            : Task.FromResult(HealthCheckResult.Unhealthy("Business rule failed"));
    }
}

// Endpoints
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Only liveness (no dependencies)
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")  // Tag dependencies
});

// K8s probes
// livenessProbe: /health/live (process alive)
// readinessProbe: /health/ready (ready for traffic)
```

#### 10. **Cost Optimization**: "Reduce Kubernetes costs by 50%"

```
1. Right-size resources (VPA - Vertical Pod Autoscaler)
   kubectl install vpa
   
2. Spot instances for fault-tolerant workloads
   nodeSelector: workload-type: batch
   tolerations: spot-instance
   
3. Cluster Autoscaler (scale nodes to 0 at night)
   
4. KEDA - Event-driven scaling (scale to 0)
   apiVersion: keda.sh/v1alpha1
   kind: ScaledObject
   metadata:
     name: myapp-keda
   spec:
     scaleTargetRef:
       name: myapp
     triggers:
     - type: cpu
       metadata:
         type: Utilization
         value: "70"
     - type: prometheus
       metadata:
         serverAddress: http://prometheus:9090
         metricName: http_requests_per_second
         threshold: "100"
     - type: kafka
       metadata:
         topic: orders
         bootstrapServers: kafka:9092
         consumerGroup: myapp
         lagThreshold: "10"

5. Clean up unused images, completed jobs, evicted pods
```

---

### GOTCHAS & EDGE CASES

- [ ] **Image Pull Secrets**: Required for private registries (ACR, ECR, GCR)
- [ ] **DNS in Cluster**: `service.namespace.svc.cluster.local` - use short names
- [ ] **Time Sync**: Containers inherit host time - ensure NTP on nodes
- [ ] **IPv6**: Disable if not needed (`sysctl -w net.ipv6.conf.all.disable_ipv6=1`)
- [ ] **Container Limits vs Requests**: Limits > Requests, or OOM on burst
- [ ] **Init Containers**: Run sequentially, must succeed before main containers
- [ ] **Pod Disruption Budget**: `minAvailable: 50%` for HA during maintenance
- [ ] **Pod Topology Spread**: Spread across zones for HA
- [ ] **Network Policies**: Default deny, explicit allow
- [ ] **Resource Quotas**: Per namespace limits

---

### CODE SNIPPETS TO MEMORIZE

#### 1. **Complete CI/CD Pipeline (GitHub Actions)**

```yaml
# .github/workflows/deploy.yaml
name: Deploy to AKS

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: myregistry.azurecr.io
  IMAGE_NAME: myapp
  AKS_CLUSTER: myapp-prod
  NAMESPACE: production

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Login to ACR
      uses: azure/docker-login@v1
      with:
        login-server: ${{ env.REGISTRY }}
        username: ${{ secrets.ACR_USERNAME }}
        password: ${{ secrets.ACR_PASSWORD }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=semver,pattern={{version}}
          type=sha
          
    - name: Build and push
      uses: docker/build-push-action@v5
      with:
        context: .
        file: src/MyApp/Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        
    - name: Run Trivy scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ steps.meta.outputs.tags }}
        severity: HIGH,CRITICAL
        exit-code: 1

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/')
    steps:
    - uses: actions/checkout@v4
    
    - name: Set up kubectl
      uses: azure/k8s-set-context@v1
      with:
        kubeconfig: ${{ secrets.KUBECONFIG }}
        
    - name: Deploy with Helm
      run: |
        helm upgrade --install myapp ./charts/myapp \
          --namespace ${{ env.NAMESPACE }} \
          --create-namespace \
          --set image.tag=${{ needs.build.outputs.image-tag }} \
          --wait --timeout 5m
          
    - name: Verify deployment
      run: |
        kubectl rollout status deployment/myapp -n ${{ env.NAMESPACE }} --timeout=300s
```

#### 2. **Kubernetes Debugging Commands**

```bash
# Resource usage
kubectl top pods -n production
kubectl top nodes

# Events
kubectl get events -n production --sort-by='.lastTimestamp'

# Describe everything
kubectl describe pod,svc,ingress,deployment,hpa -n production

# Port forward for debugging
kubectl port-forward svc/myapp 8080:80 -n production

# Exec into container
kubectl exec -it myapp-xxx -n production -- /bin/sh

# View logs with timestamps
kubectl logs -f deployment/myapp -n production --timestamps --tail=100

# Previous container logs (after crash)
kubectl logs myapp-xxx -n production --previous

# Check resource limits
kubectl get pods -n production -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].resources}{"\n"}{end}'

# Force delete stuck pod
kubectl delete pod myapp-xxx -n production --grace-period=0 --force
```

---

### PRACTICE PROBLEMS

1. **Build**: Create multi-stage Dockerfile for ASP.NET Core with: Native AOT, non-root, health checks, <80MB
2. **Deploy**: Write Helm chart with: Deployment, Service, Ingress, HPA, ServiceMonitor, PodDisruptionBudget
3. **Debug**: Pod OOMKilled - analyze memory dump, fix leak, update limits
4. **Scale**: Configure KEDA to scale on Azure Service Bus queue length
5. **Secure**: Implement Kyverno policies: no root, no latest tag, required resources, network policies

---

### ANKI FLASHCARDS

| Front | Back |
|-------|------|
| Docker Image vs Container | Image = read-only template (layers). Container = running instance + RW layer |
| Multi-stage Build Purpose | Build in SDK image, copy only artifacts to runtime image → smaller, secure |
| K8s Probe Types | Liveness: process alive. Readiness: ready for traffic. Startup: slow startup |
| RollingUpdate Strategy | maxSurge: extra pods during rollout. maxUnavailable: pods that can be down |
| Blue-Green vs Canary | Blue-Green: 2 envs, switch all. Canary: gradual % traffic to new version |
| Distroless Images | No shell, no pkg mgr, minimal CVE surface. Use for production |
| Init Container | Runs before app containers, sequentially. For migrations, config, secrets |
| Pod Disruption Budget | minAvailable/maxUnavailable during voluntary disruptions (drain, update) |
| KEDA | Event-driven autoscaling (Kafka, Azure Queue, Prometheus, CPU, etc.) |
| Graceful Shutdown | SIGTERM → stop accepting requests → finish processing → exit. preStop hook + terminationGracePeriodSeconds |

---

## NEXT TOPIC: `09-ARCHITECTURE-PATTERNS.md`

> **Study Tip**: Deploy a real ASP.NET Core app to Kubernetes (kind/k3d locally). Configure: Helm chart, HPA, Ingress with TLS, health checks, graceful shutdown, and GitHub Actions CI/CD.