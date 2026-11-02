# Finsights Platform Deployment (`finsights-platform-deployment`)

Repository chứa toàn bộ các **Helm Charts**, **Values cấu hình môi trường**, và **Khai báo Application** cho các dịch vụ ứng dụng/database/orchestration của nền tảng Finsights chạy trên cụm K3s (2 VPS Nodes).

---

## 1. Cấu trúc thư mục (Directory Structure)

```
finsights-platform-deployment/
├── .gitignore
├── README.md
├── applications/                    # Khai báo ArgoCD Application (Root App tự động kéo về)
│   ├── redis.yaml                   # Khai báo triển khai Redis HA vào cụm K3s
│   ├── postgresql.yaml              # Khai báo triển khai Shared PostgreSQL cho Tools
│   └── dagster.yaml                 # Khai báo triển khai Dagster Orchestration & Pipelines
├── charts/                          # Danh mục các Helm charts nội bộ
│   ├── redis/                       # Bitnami Redis High-Availability Chart chuẩn
│   ├── postgresql/                  # Bitnami PostgreSQL 16 Chart chuẩn (Shared Tools DB)
│   └── dagster/                     # Dagster Production Chart (Webserver, Daemon, User Code)
└── values/                          # File values tùy biến theo từng môi trường
    ├── redis-dev.yaml               # Cấu hình Standalone (tiết kiệm tài nguyên cho dev)
    ├── redis.yaml                   # Cấu hình Production Replication (HA, local-path, database-node=true)
    ├── postgresql.yaml              # Cấu hình Shared PostgreSQL (tools_db, dagster db, NodePort 30432)
    └── dagster.yaml                 # Cấu hình Dagster (kết nối PostgreSQL nội bộ, MySQL Tokyo & Singapore)
```

---

## 2. Thiết kế Hạ tầng Database Platform

### 2.1. Shared PostgreSQL (Tools Database - K8s In-Cluster)
- **Mục đích**: Lưu trữ metadata cho các công cụ điều phối, vận hành nội bộ (Dagster run/event logs, Keycloak, Grafana, v.v.).
- **Topology**: Standalone nhẹ nhàng (Requests: 100m CPU, 256Mi RAM).
- **Node Affinity**: `nodeSelector: database-node: "true"` (Node 2).
- **Persistence**: `15Gi` local-path storage.
- **Tự động khởi tạo (CoreDB)**: Tạo database `dagster` và login role `dagster_admin`; Dagster tự quản lý các bảng metadata của nó.
- **Endpoints**:
  * Internal: `postgresql.finsights-platform.svc.cluster.local:5432`
  * NodePort Tailscale: `http://100.96.150.92:30432` (kết nối DBeaver từ máy Mac).

### 2.2. Redis HA Cluster (In-Cluster)
- **Mục đích**: Cache tốc độ cao, Session store, Pub/Sub.
- **Topology**: 1 Master + 1 Replica (local-path 5Gi/pod, chạy trên Node 2).
- **Endpoint**: `redis-master.finsights-platform.svc.cluster.local:6379`.

### 2.3. Oracle Cloud Databases (External PaaS)
- **Account 1 (Tokyo - `161.33.181.173:3306`)**: **Core Website Backend Database** (Users, Auth, Portfolios, Transactions).
- **Account 2 (Singapore - `129.221.137.201:3306`)**: **AI Backend Database** (Chat sessions, Chat history, Prompt logs, Agent states).
- **Autonomous DB 23ai (`finsightsvec1` / `finsightsvec2`)**: AI Vector Search Embeddings.

---

## 3. Thiết kế Dagster Orchestration

- **Namespace**: `orchestration`.
- **Engine Storage**: Kết nối trực tiếp vào `postgresql.finsights-platform.svc.cluster.local:5432` (database: `dagster`, user: `dagster_admin`). Workflow CoreDB đồng bộ `DAGSTER_DB_PASSWORD` vào Kubernetes Secret `orchestration/dagster-admin-postgresql-secret` để chart Dagster sử dụng.
- **Run Launcher**: `K8sRunLauncher` tự tạo Job và tự giải phóng tài nguyên.
- **Web UI & Ingress**: Ingress-Nginx nhận `dagster.finsights.biz`, NodePort `31882`.
- **Pipelines**: Đọc/ghi dữ liệu nghiệp vụ vào cả 2 cụm MySQL Oracle Cloud (Tokyo và Singapore).

---

## 4. Hướng dẫn Triển khai qua GitOps (ArgoCD)

Khi bạn push code lên GitHub repo `minhphu05/finsights-platform-deployment`, ArgoCD Root Application (`root-platform-apps`) sẽ tự động quét thư mục `applications/` và đồng bộ:

```bash
git add .
git commit -m "feat: add PostgreSQL shared tools chart and update Dagster configuration"
git push origin main
```

ArgoCD sẽ tự động triển khai theo thứ tự:
1. `applications/redis.yaml` ➡️ Redis HA (namespace `finsights-platform`)
2. `applications/postgresql.yaml` ➡️ PostgreSQL Tools (namespace `finsights-platform`)
3. `applications/dagster.yaml` ➡️ Dagster Platform (namespace `orchestration`)
