# Finsights Platform Deployment (`finsights-platform-deployment`)

Repository chứa toàn bộ các **Helm Charts**, **Values cấu hình môi trường**, và **Khai báo Application** cho các dịch vụ ứng dụng/database của nền tảng Finsights chạy trên cụm K3s (2 VPS Nodes).

---

## 1. Cấu trúc thư mục (Directory Structure)

```
finsights-platform-deployment/
├── .gitignore
├── README.md
├── applications/                    # Khai báo ArgoCD Application (Root App sẽ tự động kéo về)
│   └── redis.yaml                   # Khai báo triển khai Redis vào cụm K3s
├── charts/                          # Danh mục các Helm charts nội bộ
│   └── redis/                       # Bitnami Redis High-Availability Chart chuẩn
│       ├── Chart.yaml
│       ├── charts/common/           # Subchart bitnami-common
│       ├── templates/
│       └── values.yaml
└── values/                          # File values tùy biến theo từng môi trường
    ├── redis-dev.yaml               # Cấu hình Standalone (tiết kiệm tài nguyên cho dev)
    └── redis.yaml                   # Cấu hình Production Replication (HA, local-path, database-node=true)
```

---

## 2. Thiết kế Helm Chart Redis cho Finsights

### Đặc tả kỹ thuật (Production `values/redis.yaml`):
- **Topology**: Replication (1 Master + 1 Replica).
- **Node Affinity / Scheduling**: Chỉ định `nodeSelector: database-node: "true"` để ép các Pod Redis chạy trên **Node 2** (`103.249.117.201`) – node chuyên trách lưu trữ Database.
- **Persistence (Lưu trữ bền vững)**:
  - `storageClass: local-path` (StorageClass mặc định của K3s).
  - Dung lượng: `5Gi` cho Master và `5Gi` cho Replica.
- **Bảo mật & Authentication**:
  - `auth.enabled: true`
  - Mật khẩu: `FinsightsRedis2026!`
- **Observability / Giám sát**:
  - Tích hợp sẵn `metrics.enabled: true` (Prometheus `redis-exporter` chạy kèm trên port `9121`).
  - Sẵn sàng để OCI Management Agent scrape metrics tự động.
- **Tài nguyên tối ưu cho VPS**:
  - Master requests: `100m CPU`, `128Mi RAM` (limits: `500m CPU`, `1Gi RAM`).
  - Replica requests: `50m CPU`, `128Mi RAM` (limits: `250m CPU`, `512Mi RAM`).

---

## 3. Hướng dẫn Triển khai (Deployment Instructions)

### Cách 1: Triển khai qua ArgoCD GitOps (Khuyên dùng)
Khi đã bootstrap ArgoCD từ repo `finsights-gitops-infra`, ArgoCD Root Application (`root-platform-apps`) sẽ tự động phát hiện file `applications/redis.yaml` và đồng bộ vào cụm K3s.

Nếu muốn áp dụng ngay lập tức:
```bash
kubectl apply -f applications/redis.yaml
```

### Cách 2: Triển khai trực tiếp bằng Helm CLI (Manual)
```bash
# Tạo namespace nếu chưa có
kubectl create namespace finsights-platform

# Cài đặt bằng Helm
helm upgrade --install redis charts/redis \
  -f values/redis.yaml \
  -n finsights-platform
```

---

## 4. Hướng dẫn Kết nối & Kiểm tra (Verification)

### 4.1. Thông tin Endpoint nội bộ K8s
- **Master Service (Ghi & Đọc)**: `redis-master.finsights-platform.svc.cluster.local:6379`
- **Replicas Service (Chỉ Đọc)**: `redis-replicas.finsights-platform.svc.cluster.local:6379`

### 4.2. Chạy Pod kiểm tra `redis-cli`:
```bash
kubectl run --namespace finsights-platform redis-client --rm --tty -i --restart='Never' \
  --image docker.io/bitnami/redis:latest \
  --command -- redis-cli -h redis-master -a 'FinsightsRedis2026!'
```

### 4.3. Kiểm tra Metrics Prometheus Endpoint:
```bash
kubectl port-forward svc/redis-metrics 9121:9121 -n finsights-platform
curl http://localhost:9121/metrics
```
