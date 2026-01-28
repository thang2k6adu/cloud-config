## 1. K3s Cluster (Kubernetes)

* Có **1 Master + nhiều Worker**
* Pod được schedule tự động giữa các node
* Có label role worker
* Quản lý bằng `kubectl`
* Scale, rollout, rollback workload được
---

## 2. Networking nội bộ an toàn bằng WireGuard VPN

* VPS ↔ Master ↔ Workers kết nối qua mạng riêng: `10.10.10.0/24`
* Không expose trực tiếp node LAN ra Internet
* Traffic từ Internet → VPS → VPN → Cluster
---

## 3. Ingress Controller (NGINX Ingress)

* Dùng `ingress-nginx` thay Traefik
* Route theo domain:

  * `dashboard.domain`
  * `grafana.domain`
  * `nginx.domain`
* Không cần NodePort cho từng app
* Chuẩn Kubernetes Ingress

---

## 4. Nginx Reverse Proxy ngoài VPS (Public)

* VPS chạy:

  * Nginx
  * Certbot (SSL)
  * Rate limit
  * Security headers
* Proxy về ingress-nginx qua WireGuard
* Có script `add-domain` để:

  * Tạo domain mới
  * Auto SSL
  * Reload nginx

---

## 5. HTTPS chuẩn (Let’s Encrypt)

* Auto cấp chứng chỉ SSL cho từng domain
* Redirect HTTP → HTTPS
* HSTS + security headers

---

## 6. Monitoring stack

Bạn đã cài:

* Prometheus
* Grafana
* Alertmanager
* Node-exporter

Theo dõi:

* CPU
* RAM
* Disk
* Pod
* Node
* Cluster

---

## 7. Kubernetes Dashboard

* Truy cập qua domain
* Login bằng token
* Quản lý cluster bằng web UI

---
