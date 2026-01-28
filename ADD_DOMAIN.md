# Cấu Trúc Nginx Configuration

## Cấu trúc thư mục

```
/etc/nginx/nginx.conf        (KHÔNG ĐỘNG)

/etc/nginx/backends/
    ingress.conf

/etc/nginx/conf.d/
    ingress_upstream.conf
    security.conf
    rate_limit.conf
    gzip.conf
    cache.conf

/etc/nginx/sites-available/
    kruzetech.dev

/etc/nginx/sites-enabled/
    kruzetech.dev -> ../sites-available/kruzetech.dev
```

---

## Setup trên VPS

### 1. Cài đặt Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

---

### 2. Lấy IP của tất cả các node (workers:master)

**Cài jq:** trên master
```bash
sudo apt update
sudo apt install -y jq
```

**Lấy IP VPN:** trên master
```bash
ansible-inventory -i ~/k3s-inventory/hosts.ini --list \
| jq -r '
._meta.hostvars
| to_entries[]
| select(.value.ansible_user=="thang2k6adu")
| "server \(.value.vpn_ip):30443;"
'
```

**Phải ra:**
```nginx
    server 10.10.10.11:30080;
    server 10.10.10.13:30080;
    server 10.10.10.12:30080;
```

---

### 3. Tạo backend list riêng

```bash
sudo mkdir -p /etc/nginx/backends
sudo nano /etc/nginx/backends/ingress.conf
```

**Nội dung `/etc/nginx/backends/ingress.conf`:**
```nginx
server 10.10.10.11:30443;
server 10.10.10.12:30443;
server 10.10.10.13:30443;
```

---

### 4. Tạo upstream Global

```bash
sudo nano /etc/nginx/conf.d/ingress_upstream.conf
```

**Nội dung `/etc/nginx/conf.d/ingress_upstream.conf`:**
```nginx
upstream ingress_http {
    least_conn;
    include /etc/nginx/backends/ingress.conf;
}
```

---

### 5. Tạo security global

```bash
sudo nano /etc/nginx/conf.d/security.conf
```

**Nội dung `/etc/nginx/conf.d/security.conf`:**
```nginx
server_tokens off;

add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy strict-origin-when-cross-origin always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

---

### 6. Rate limit

```bash
sudo nano /etc/nginx/conf.d/rate_limit.conf
```

**Nội dung `/etc/nginx/conf.d/rate_limit.conf`:**
```nginx
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
```

---

## Tạo script add domain

### Cấu hình domain mẫu

1 domain phải như này:

```nginx
server {
    listen 80;
    server_name kruzetech.dev www.kruzetech.dev;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name kruzetech.dev www.kruzetech.dev;

    ssl_certificate /etc/letsencrypt/live/kruzetech.dev/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kruzetech.dev/privkey.pem;

    location / {
        proxy_pass https://ingress_http;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_ssl_server_name on;
    }

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass https://ingress_http;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_ssl_server_name on;
    }
}
```

---

### Tạo script

```bash
sudo nano /usr/local/bin/add-domain
sudo chmod +x /usr/local/bin/add-domain
```
cài certbot
sudo apt install -y certbot python3-certbot-nginx

**Nội dung `/usr/local/bin/add-domain`:**
```bash
#!/bin/bash

DOMAIN=$1

if [ -z "$DOMAIN" ]; then
  echo "Usage: add-domain domain.com"
  exit 1
fi

CONF="/etc/nginx/sites-available/$DOMAIN"

if [ -f "$CONF" ]; then
  echo "Domain already exists: $DOMAIN"
  exit 1
fi

cat > $CONF <<EOF
server {
    listen 80;
    server_name $DOMAIN www.$DOMAIN;

    location / {
        proxy_pass http://ingress_http;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    }
}
EOF

ln -s $CONF /etc/nginx/sites-enabled/$DOMAIN

nginx -t || exit 1
systemctl reload nginx

# Xin cert
certbot --nginx -d $DOMAIN -d www.$DOMAIN --non-interactive --agree-tos -m admin@$DOMAIN

cat > $CONF <<EOF
server {
    listen 80;
    server_name $DOMAIN www.$DOMAIN;
    return 301 https://$DOMAIN\$request_uri;
}

server {
    listen 443 ssl http2;
    server_name www.$DOMAIN;

    ssl_certificate /etc/letsencrypt/live/$DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;

    return 301 https://$DOMAIN\$request_uri;
}

server {
    listen 443 ssl http2;
    server_name $DOMAIN;

    ssl_certificate /etc/letsencrypt/live/$DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;

    location / {
        proxy_pass https://ingress_http;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_ssl_server_name on;
    }

    location /api/ {
        limit_req zone=api_limit burst=20 nodelay;

        proxy_pass https://ingress_http;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_ssl_server_name on;
    }
}
EOF

nginx -t || exit 1
systemctl reload nginx
```

---

## Các thao tác quản lý

### Thêm domain

```bash
sudo add-domain dashboard.kruzetech.dev
```

**Lưu ý:** thêm domain thì phải thêm www. nữa nhé

---

### Thêm node backend

```bash
echo "server 10.10.10.14:30443;" >> /etc/nginx/backends/ingress.conf
nginx -t && systemctl reload nginx
```

---

### Remove domain

```bash
sudo rm -f /etc/nginx/sites-enabled/dashboard.kruzetech.dev
sudo rm -f /etc/nginx/sites-available/dashboard.kruzetech.dev
sudo certbot delete --cert-name dashboard.kruzetech.dev
sudo nginx -t && sudo systemctl reload nginx
```

lên master node thêm cái để test

thêm ingress cho dashboard bên K3S

check
kubectl get svc -n kubernetes-dashboard

tạo
nano ~/k8s-manifest/dashboard-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  rules:
  - host: dashboard.kruzetech.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443


apply
kubectl apply -f ~/k8s-manifest/dashboard-ingress.yaml

check
kubectl get ingress -n kubernetes-dashboard

lấy token
kubectl -n kubernetes-dashboard create token kubernetes-dashboard-admin

đăng nhập dashboard với token trên

tạo ingress grafana
nano ~/k8s-manifest/grafana-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
spec:
  ingressClassName: nginx
  rules:
  - host: grafana.kruzetech.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: monitoring-grafana
            port:
              number: 80

apply
kubectl apply -f ~/k8s-manifest/grafana-ingress.yaml

check 
kubectl get ingress -n monitoring

add domain

luồng chuẩn
tạo pod -> service -> ingress -> nginx reverse proxy (add domain) -> internets