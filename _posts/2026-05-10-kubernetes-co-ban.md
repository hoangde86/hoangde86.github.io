---
title: "Kubernetes cho người mới bắt đầu: Từ Pod đến Deployment"
date: 2026-05-10 09:00:00 +0700
categories: [DevOps, Kubernetes]
tags: [kubernetes, k8s, container, devops]
---

Kubernetes (K8s) là nền tảng orchestration container phổ biến nhất hiện nay. Bài này tập trung vào những khái niệm cốt lõi bạn cần nắm trước.

## Kiến trúc tổng quan

```
Control Plane                    Worker Nodes
┌─────────────────┐              ┌──────────────┐
│  API Server     │◄────────────►│  kubelet     │
│  etcd           │              │  kube-proxy  │
│  Scheduler      │              │  container   │
│  Controller Mgr │              │  runtime     │
└─────────────────┘              └──────────────┘
```

## Các object cơ bản

### Pod — Đơn vị nhỏ nhất

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "100m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

> Không bao giờ tạo Pod trực tiếp trong production — dùng Deployment.

### Deployment — Quản lý Pod tự động

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

### Service — Expose ứng dụng

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP   # hoặc NodePort, LoadBalancer
```

## Các lệnh kubectl thường dùng

```bash
# Xem trạng thái
kubectl get pods -n default
kubectl get pods -A                    # tất cả namespace
kubectl describe pod <name>            # chi tiết
kubectl logs -f <pod-name>             # xem log

# Thao tác
kubectl apply -f deployment.yml        # tạo/cập nhật
kubectl delete -f deployment.yml       # xóa
kubectl scale deployment nginx --replicas=5

# Debug
kubectl exec -it <pod> -- bash         # vào trong pod
kubectl port-forward pod/<name> 8080:80  # tunnel về local
kubectl top pods                        # xem CPU/RAM usage
```

## ConfigMap và Secret

```yaml
# ConfigMap cho config không nhạy cảm
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres-service"
  LOG_LEVEL: "info"

---
# Secret cho thông tin nhạy cảm
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DATABASE_PASSWORD: "mysecretpassword"
```

Dùng trong Deployment:

```yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: app-secret
```

## Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx-deployment
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

Tự động scale khi CPU vượt 70% — giảm chi phí khi traffic thấp, tăng capacity khi traffic cao.

---

Phần tiếp theo sẽ đề cập **Ingress, Persistent Volumes và Helm**.
