# 24 - CI/CD, Secrets Management & Kubernetes Deployment

> **Author:** Emerson Lima — [github.com/Emersondll](https://github.com/Emersondll)
>
> Mandatory standards for CI/CD pipelines, secrets handling, containerization, and Kubernetes/ArgoCD deployments.
> Every microservice MUST follow this document before any production deployment.

---

## 1. Core Security Rules

- **NEVER** hardcode credentials, API keys, tokens, or passwords in source code
- **NEVER** commit `.env` files to git — they MUST be in `.gitignore`
- **NEVER** put plain secrets in `application.yml` directly — always use `${ENV_VAR}` syntax
- **NEVER** print or log secrets — not even masked versions in dev logs
- All credentials are rotated immediately on suspected exposure — zero tolerance
- Secrets live in Vault; environment-specific values injected via Kubernetes Secrets or ArgoCD

---

## 2. `.env` Files — Local Development Only

### `.env` structure

```bash
# .env — NEVER commit this file
MONGODB_URI=mongodb://localhost:27017/app_dev
RABBITMQ_HOST=localhost
RABBITMQ_USER=guest
RABBITMQ_PASSWORD=guest
GATEWAY_TOKEN=local-dev-gateway-token
JWT_SECRET=local-dev-jwt-secret-min-32-chars
```

### `.gitignore` mandatory entries

```gitignore
# Secrets — never committed
.env
.env.local
.env.*.local
*.env

# Secrets files
secrets/
vault-token
```

### Loading `.env` in Spring Boot locally

```yaml
# application-dev.yml
spring:
  config:
    import: optional:file:.env[.properties]
```

> Alternatively, add the `spring-dotenv` library (`me.paulschwarz:spring-dotenv`) for native `.env` support.

---

## 3. `application.yml` — Secrets via Environment Variables Only

```yaml
# CORRECT — env var references; no defaults in prod
spring:
  data:
    mongodb:
      uri: ${MONGODB_URI}             # Required — no default
  rabbitmq:
    password: ${RABBITMQ_PASSWORD}    # Required — no default

security:
  gateway:
    token: ${GATEWAY_TOKEN}           # Required — no default

jwt:
  secret: ${JWT_SECRET}               # Required — no default

# WRONG — never do this
spring:
  data:
    mongodb:
      uri: mongodb://user:password@prod-host:27017/db   # FORBIDDEN
```

**Rule:** If a key has no `${ENV_VAR}` reference in `application-prod.yml`, it must not exist there at all. Missing variables fail fast at startup (see `19-SPRING-PROFILES.md` fail-fast bean).

---

## 4. GitHub Actions — CI Pipeline

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  JAVA_VERSION: '26'
  MAVEN_OPTS: '-Xmx1024m'

jobs:
  build-and-test:
    name: Build, Test & Coverage
    runs-on: ubuntu-latest

    # For MongoDB projects:
    services:
      mongodb:
        image: mongo:7.0
        ports:
          - 27017:27017
    # For JPA/PostgreSQL projects, remove the mongodb service above and use TestContainers
    # in the test code itself (@DynamicPropertySource) — no separate services: block needed.

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Java ${{ env.JAVA_VERSION }}
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'

      - name: Build and verify
        run: ./mvnw verify
        env:
          MONGODB_URI: mongodb://localhost:27017/test
          GATEWAY_TOKEN: ci-gateway-token
          JWT_SECRET: ci-jwt-secret-for-testing-only-32-chars

      - name: Upload JaCoCo report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: jacoco-report
          path: target/site/jacoco/

      - name: SpotBugs check
        run: ./mvnw spotbugs:check

      - name: Secret leak scan
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### GitHub Secrets (repository level)

Store in **GitHub → Settings → Secrets and Variables → Actions**. Never inline in YAML.

| Secret Name | Purpose |
|---|---|
| `DOCKER_USERNAME` | Docker Hub or registry username |
| `DOCKER_PASSWORD` | Docker Hub or registry access token |
| `SONAR_TOKEN` | SonarQube analysis token |
| `KUBE_CONFIG` | Base64-encoded kubeconfig (for deploy jobs) |

Reference in workflow YAML: `${{ secrets.DOCKER_USERNAME }}`

---

## 5. GitHub Actions — CD Pipeline (Build & Push Docker Image)

### `.github/workflows/cd.yml`

```yaml
name: CD

on:
  push:
    branches: [ main ]
    tags: [ 'v*.*.*' ]

jobs:
  docker-build-push:
    name: Build & Push Docker Image
    runs-on: ubuntu-latest
    needs: [ build-and-test ]    # Only runs if CI passed

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Registry
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: emersondll/user-service
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: emersondll/user-service:${{ github.sha }}
          format: 'table'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Update image tag in GitOps repo
        run: |
          sed -i "s|newTag:.*|newTag: ${{ github.sha }}|" \
            infra/k8s/overlays/prod/kustomization.yaml
          git config user.email "ci@github.com"
          git config user.name "GitHub Actions"
          git add infra/k8s/overlays/prod/kustomization.yaml
          git commit -m "chore(deploy): update user-service to ${{ github.sha }}"
          git push
```

---

## 6. Dockerfile — Multi-Stage Build

```dockerfile
# Build stage
FROM maven:3.9.9-eclipse-temurin-26-alpine AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline -q
COPY src ./src
RUN mvn package -DskipTests -q

# Runtime stage
FROM eclipse-temurin:26-jre-alpine AS runtime
WORKDIR /app

# Non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/target/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", \
    "-server", \
    "-XX:+UseG1GC", \
    "-XX:MaxGCPauseMillis=200", \
    "-XX:+ExitOnOutOfMemoryError", \
    "-Dfile.encoding=UTF-8", \
    "-jar", "app.jar"]
```

**Rules:**
- Always run as a non-root user
- Never include source code or `.env` in the final runtime image
- Pin base image versions — never use `latest`

---

## 7. HashiCorp Vault — Secrets Management

### Why Vault

| Feature | Benefit |
|---|---|
| Centralized storage | One source of truth for all microservices |
| Dynamic secrets | MongoDB/PostgreSQL credentials rotate automatically |
| Audit trail | Every secret access is logged |
| Least privilege | Per-service policies limit blast radius |

### Spring Boot + Vault integration

Add to `pom.xml` (version managed by Spring Cloud BOM — see `20-MAVEN-POM.md`):
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
<!-- Required to enable bootstrap context in Spring Boot 4 -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
</dependency>
```

```yaml
# bootstrap.yml (loaded before application context — requires spring-cloud-starter-bootstrap)
spring:
  cloud:
    vault:
      host: ${VAULT_HOST:vault}
      port: ${VAULT_PORT:8200}
      scheme: https
      authentication: KUBERNETES      # Uses K8s service account token
      kubernetes:
        role: ${VAULT_ROLE:user-service}
        kubernetes-path: kubernetes
      kv:
        enabled: true
        backend: secret
        application-name: user-service
```

### Vault secret path convention

```
secret/
  microservices/
    <service-name>/
      mongodb-uri
      rabbitmq-password
      jwt-secret
      gateway-token
```

### Vault policy per service (HCL)

```hcl
# policy: user-service
path "secret/data/microservices/user-service/*" {
  capabilities = ["read"]
}
```

### Enabling K8s auth in Vault

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
vault write auth/kubernetes/role/user-service \
    bound_service_account_names=user-service \
    bound_service_account_namespaces=production \
    policies=user-service \
    ttl=24h
```

---

## 8. Kubernetes Secrets

### When to use without Vault

For simpler environments. **Always enable encryption at rest** (`EncryptionConfiguration`) when storing credentials as K8s Secrets.

```yaml
# k8s/secrets.yaml — NEVER commit with real values; use placeholders + CI substitution
apiVersion: v1
kind: Secret
metadata:
  name: user-service-secrets
  namespace: production
type: Opaque
stringData:
  MONGODB_URI: "placeholder"        # Replaced by CI/CD pipeline at deploy time
  GATEWAY_TOKEN: "placeholder"
  JWT_SECRET: "placeholder"
```

### Reference in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      serviceAccountName: user-service    # Required for Vault K8s auth
      containers:
        - name: user-service
          image: emersondll/user-service:v1.2.0
          ports:
            - containerPort: 8080
          envFrom:
            - secretRef:
                name: user-service-secrets
            - configMapRef:
                name: user-service-config
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 15
            failureThreshold: 3
```

---

## 9. ArgoCD — GitOps Deployment

### What is ArgoCD

ArgoCD is a declarative GitOps CD tool for Kubernetes. It treats Git as the single source of truth: the cluster is continuously reconciled to match what's in the repository.

### Repository structure for GitOps

```
infra/
  k8s/
    base/
      deployment.yaml
      service.yaml
      configmap.yaml
      kustomization.yaml
    overlays/
      dev/
        kustomization.yaml      # patch: 1 replica, dev image tag
        patch-replicas.yaml
      staging/
        kustomization.yaml
      prod/
        kustomization.yaml      # patch: 2+ replicas, prod image tag
        patch-replicas.yaml
        patch-resources.yaml
```

### ArgoCD Application manifest

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: user-service
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Emersondll/user-service
    targetRevision: main
    path: infra/k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true        # Remove K8s resources deleted from Git
      selfHeal: true     # Revert manual cluster changes to match Git
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

### Full CD flow with ArgoCD

```
1. Developer merges PR to main (branch protection enforced)
       ↓
2. GitHub Actions: ./mvnw verify (tests + JaCoCo gate)
       ↓
3. GitHub Actions: docker build + push → registry:sha-abc123
       ↓
4. GitHub Actions: Trivy vulnerability scan (fails on CRITICAL/HIGH)
       ↓
5. GitHub Actions: update newTag in infra/k8s/overlays/prod/kustomization.yaml → commit
       ↓
6. ArgoCD detects Git change (polling or webhook)
       ↓
7. ArgoCD syncs cluster: rolling update (zero-downtime)
       ↓
8. Kubernetes: readinessProbe gates traffic until new pods are healthy
       ↓
9. ArgoCD reports Synced + Healthy
```

### `kustomization.yaml` with image pinning

```yaml
# infra/k8s/overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
images:
  - name: emersondll/user-service
    newTag: sha-abc123def456    # Updated by CI/CD pipeline
```

---

## 10. Kubernetes ConfigMap (Non-Secret Configuration)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-service-config
  namespace: production
data:
  SERVER_PORT: "8080"
  MANAGEMENT_ENDPOINTS_WEB_EXPOSURE_INCLUDE: "health,info,metrics,prometheus"
  MANAGEMENT_TRACING_SAMPLING_PROBABILITY: "0.1"
  SPRING_JACKSON_DESERIALIZATION_FAIL_ON_NULL_FOR_PRIMITIVES: "false"
```

**Rule:** ConfigMap = non-sensitive configuration. Secret = passwords, tokens, connection strings with credentials.

---

## 11. Sealed Secrets — Encrypting Secrets for Git

For teams that need to store secret manifests in Git:

```bash
# Install kubeseal CLI and fetch cluster cert
kubeseal --fetch-cert --controller-namespace=kube-system > pub-cert.pem

# Encrypt secrets.yaml → sealed-secrets.yaml (safe to commit)
kubeseal --format=yaml --cert=pub-cert.pem \
  < k8s/secrets.yaml \
  > k8s/sealed-secrets.yaml
```

`sealed-secrets.yaml` can be safely committed — only the cluster's Sealed Secrets controller can decrypt it. The original `secrets.yaml` must remain in `.gitignore`.

---

## 12. Security Checklist

### Repository level
- [ ] `.env` and `*.env` entries in `.gitignore`
- [ ] No credentials in `application.yml`, `application-*.yml`, or `pom.xml`
- [ ] GitHub branch protection enabled on `main` (require PR + CI pass)
- [ ] GitHub Secrets configured: `DOCKER_USERNAME`, `DOCKER_PASSWORD`, `SONAR_TOKEN`, `KUBE_CONFIG`
- [ ] `gitleaks` step present in CI to fail on secret leaks
- [ ] Secret scanning enabled in GitHub Advanced Security settings

### CI/CD level
- [ ] All secrets referenced via `${{ secrets.X }}` — never hardcoded in workflow YAML
- [ ] CI fails if JaCoCo coverage gate < 80%
- [ ] CD blocked until CI passes (`needs: [build-and-test]`)
- [ ] Trivy scan blocks push on `CRITICAL` or `HIGH` vulnerabilities
- [ ] Docker image runs as non-root user (`USER appuser`)
- [ ] Base images pinned to specific versions — never `latest`

### Kubernetes level
- [ ] Secrets mounted via `envFrom.secretRef` (not `env[].value` inline)
- [ ] `resources.requests` and `resources.limits` defined on every container
- [ ] `readinessProbe` and `livenessProbe` configured on every pod
- [ ] `serviceAccountName` set (required for Vault K8s auth)
- [ ] Namespace isolation: `dev`, `staging`, `prod` in separate namespaces
- [ ] NetworkPolicy restricting inter-service traffic to minimum required

### Vault level (when using Vault)
- [ ] Kubernetes auth method active — no static Vault tokens in cluster
- [ ] Per-service policy with least-privilege access
- [ ] Vault audit log enabled and shipped to SIEM
- [ ] Dynamic secrets configured for MongoDB/PostgreSQL (auto-rotation)
- [ ] Vault HA configured — not single-node in production

### ArgoCD level
- [ ] `selfHeal: true` to revert unauthorized manual cluster changes
- [ ] `prune: true` to remove resources deleted from Git
- [ ] ArgoCD access restricted (SSO + RBAC policies)
- [ ] Notifications configured for sync failures

---

## Related Specs

| Spec | Relationship |
|---|---|
| `23-GIT-BRANCHING.md` | Branch protection and PR workflow that triggers CI/CD |
| `20-MAVEN-POM.md` | Plugin config for what `./mvnw verify` actually runs (JaCoCo, SpotBugs, Failsafe) |
| `06-DOCKER-DEPLOYMENT.md` | Full Docker and Docker Compose spec; this file focuses on CI/CD usage of Docker |
| `19-SPRING-PROFILES.md` | Spring profiles loaded at runtime; secrets injected via env vars described here |
| `14-SECURITY.md` | Trust boundary and `GatewayAuthFilter` — `GATEWAY_TOKEN` secret managed here |
| `18-MONGODB-INDEXES.md` | `auto-index-creation: false` in prod — indexes via Mongock, not startup |

---

*Maintained by [Emerson Lima](https://github.com/Emersondll)*
