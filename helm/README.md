# RAGFlow Helm Chart

A Helm chart to deploy RAGFlow and its dependencies on Kubernetes.

- Components: RAGFlow (web/api) and optional dependencies (Infinity/Elasticsearch/OpenSearch, MySQL, MinIO, Redis)
- Requirements: Kubernetes >= 1.24, Helm >= 3.10

## Install

```bash
helm upgrade --install ragflow ./ \
  --namespace ragflow --create-namespace
```

Uninstall:
```bash
helm uninstall ragflow -n ragflow
```

## Global Settings

- `global.repo`: Prepend a global image registry prefix for all images.
  - Behavior: Replaces the registry part and keeps the image path (e.g., `quay.io/minio/minio` -> `registry.example.com/myproj/minio/minio`).
  - Example: `global.repo: "registry.example.com/myproj"`
- `global.imagePullSecrets`: List of image pull secrets applied to all Pods.
  - Example:
    ```yaml
    global:
      imagePullSecrets:
        - name: regcred
    ```

## External Services (MySQL / MinIO / Redis)

The chart can deploy in-cluster services or connect to external ones. Toggle with `*.enabled`. When disabled, provide host/port via `env.*`.

- MySQL
  - `mysql.enabled`: default `true`
  - If `false`, set:
    - `env.MYSQL_HOST` (required), `env.MYSQL_PORT` (default `3306`)
    - `env.MYSQL_DBNAME` (default `rag_flow`), `env.MYSQL_PASSWORD` (required)
    - `env.MYSQL_USER` (default `root` if omitted)
- MinIO
  - `minio.enabled`: default `true`
  - Configure:
    - `env.MINIO_HOST` (optional external host), `env.MINIO_PORT` (default `9000`)
    - `env.MINIO_ROOT_USER` (default `rag_flow`), `env.MINIO_PASSWORD` (optional)
- Redis (Valkey)
  - `redis.enabled`: default `true`
  - If `false`, set:
    - `env.REDIS_HOST` (required), `env.REDIS_PORT` (default `6379`)
    - `env.REDIS_PASSWORD` (optional; empty disables auth if server allows)

Notes:
- When `*.enabled=true`, the chart renders in-cluster resources and injects corresponding `*_HOST`/`*_PORT` automatically.
- Sensitive variables like `MYSQL_PASSWORD` are required; `MINIO_PASSWORD` and `REDIS_PASSWORD` are optional. All secrets are stored in a Secret.

### Example: use external MySQL, MinIO, and Redis

```yaml
# values.override.yaml
mysql:
  enabled: false  # use external MySQL
minio:
  enabled: false  # use external MinIO (S3 compatible)
redis:
  enabled: false  # use external Redis/Valkey

env:
  # MySQL
  MYSQL_HOST: mydb.example.com
  MYSQL_PORT: "3306"
  MYSQL_USER: root
  MYSQL_DBNAME: rag_flow
  MYSQL_PASSWORD: "<your-mysql-password>"

  # MinIO
  MINIO_HOST: s3.example.com
  MINIO_PORT: "9000"
  MINIO_ROOT_USER: rag_flow
  MINIO_PASSWORD: "<your-minio-secret>"

  # Redis
  REDIS_HOST: redis.example.com
  REDIS_PORT: "6379"
  REDIS_PASSWORD: "<your-redis-pass>"
```

Apply:
```bash
helm upgrade --install ragflow ./helm -n ragflow -f values.override.yaml
```

## Document Engine Selection

Choose one of `infinity` (default), `elasticsearch`, or `opensearch` via `env.DOC_ENGINE`. The chart renders only the selected engine and sets the appropriate host variables.

```yaml
env:
  DOC_ENGINE: infinity   # or: elasticsearch | opensearch
  # For elasticsearch
  ELASTIC_PASSWORD: "<es-pass>"
  # For opensearch
  OPENSEARCH_PASSWORD: "<os-pass>"
```

## Ingress

Expose the web UI via Ingress:

```yaml
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: ragflow.example.com
      paths:
        - path: /
          pathType: Prefix
```

## Validate the Chart

```bash
helm lint ./helm
helm template ragflow ./helm > rendered.yaml
```

## Notes

- By default, the chart uses `DOC_ENGINE: infinity` and deploys in-cluster MySQL, MinIO, and Redis.
- The chart injects derived `*_HOST`/`*_PORT` and required secrets into a single Secret (`<release>-ragflow-env-config`).
- `global.repo` and `global.imagePullSecrets` apply to all Pods; per-component `*.image.pullSecrets` still work and are merged with global settings.

## Autoscaling (HPA) for API/Worker

The chart supports Kubernetes HPA (`autoscaling/v2`) for `ragflow-api` and `ragflow-worker` targets.

```yaml
ragflow:
  replicaCount: 1

  api:
    deploymentName: ""   # default: <release-name>-api
    autoscaling:
      enabled: false
      minReplicas: 1
      maxReplicas: 3
      targetCPUUtilizationPercentage: 70
      customMetrics: []

  worker:
    deploymentName: ""   # default: <release-name>-worker
    autoscaling:
      enabled: false
      minReplicas: 1
      maxReplicas: 5
      targetCPUUtilizationPercentage: 75
      customMetrics: []

  admin:
    replicaCount: 1
    autoscaling:
      enabled: false
```

- `ragflow-admin` defaults to **HPA disabled** and **fixed 1 replica**.
- `customMetrics` is passed directly into HPA `spec.metrics`, so you can add Pods/Object/External metrics (for example: RPS or queue depth) when metrics adapters are installed.

### 운영 정책: 수동 scale vs HPA 우선순위

- HPA가 활성화된 워크로드(`ragflow.api.autoscaling.enabled=true`, `ragflow.worker.autoscaling.enabled=true`)는 HPA가 `spec.replicas`를 지속적으로 조정하므로, `kubectl scale deployment ...`로 수동 변경한 값은 일시적이다.
- 운영 표준 우선순위는 **HPA 설정값(min/max/metrics) > 수동 scale** 로 둔다.
- 긴급 대응으로 수동 scale이 필요하면:
  1. 해당 워크로드 HPA를 일시 비활성화하거나 삭제
  2. 수동 scale 적용
  3. 사후에 HPA 정책 재적용

### 운영 정책: 장애 시 failover

- HPA는 장애 복구(Failover) 메커니즘이 아니라 **부하 기반 복제 수 조절** 기능이다.
- 장애 대응은 아래 계층에서 수행한다.
  - **Pod 수준**: readiness/liveness probe와 Deployment rolling update/재시작.
  - **노드 수준**: PodDisruptionBudget, 다중 노드 스케줄링(anti-affinity/zone spread)로 단일 노드 장애 영향 최소화.
  - **서비스 수준**: Service가 Ready Pod로만 트래픽 라우팅.
- 권장 정책:
  - API/Worker는 최소 2 replicas(운영 환경) + PDB 적용.
  - Admin은 기본 1 replica 유지하되, 장애 민감 환경에서는 별도 관리 플레인 이중화 전략을 사용.
