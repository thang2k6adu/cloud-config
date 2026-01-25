# 🚀 HƯỚNG DẪN TRIỂN KHAI K3S CLUSTER (MASTER + WORKER)

---

## BƯỚC 1: ĐỔI HOSTNAME (TRÊN NODE MASTER)

> Nhớ dùng `ip a` để check **IP / mask / gateway** và thay cho đúng trước khi làm bất cứ điều gì.

```bash
sudo hostnamectl set-hostname k3s-master
sudo nano /etc/hosts
```

Ví dụ nội dung:

```txt
127.0.0.1 localhost
192.168.0.104 k3s-master
```

Reboot:

```bash
sudo reboot
```

---

## BƯỚC 2: SET IP TĨNH + DISABLE CLOUD-INIT (MASTER)

Disable cloud-init network:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Nội dung:

```yaml
network: {config: disabled}
```

Xóa netplan cũ:

```bash
sudo rm -f /etc/netplan/50-cloud-init.yaml
```

Tạo netplan mới:

```bash
sudo nano /etc/netplan/01-static.yaml
```

Nội dung:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.0.104/24
      gateway4: 192.168.0.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Apply:

```bash
sudo netplan apply
```

Check IP:

```bash
ip a
```

---

## BƯỚC 3: SCAN IP CÁC SERVER WORKER (TRÊN MASTER)

Cài `nmap`:

```bash
sudo apt install nmap -y
```

Auto generate inventory file

> ⚠️ Nhớ sửa subnet + port SSH cho đúng môi trường
sau này thêm server thì nhớ chạy lại cái này là oke

```bash
SUBNET="192.168.0.0/24"
PORT=8022
USER="thang2k6adu"
MASTER_IP=$(hostname -I | awk '{print $1}')

mkdir -p ~/k3s-inventory && cd ~/k3s-inventory

echo -e "[master]\n$MASTER_IP ansible_user=$USER ansible_port=$PORT worker_ip=$MASTER_IP\n\n[workers]" > hosts.ini

sudo nmap -p $PORT --open $SUBNET \
| grep "Nmap scan report" \
| grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" \
| grep -v "$MASTER_IP" \
| sed "s/.*/& ansible_user=$USER ansible_port=$PORT worker_ip=&/" \
>> hosts.ini

cd ~/
```

Check file inventory:

```bash
cat ~/k3s-inventory/hosts.ini
```

Kết quả mong đợi:

```ini
[master]
192.168.0.104 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.104

[workers]
192.168.0.105 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.105
192.168.0.106 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.106
```

---

## BƯỚC 4: CÀI K3S CONTROL PLANE (MASTER)

Đặt tên node là `k3s-master`:

```bash
curl -sfL https://get.k3s.io | sh -s - \
  --write-kubeconfig-mode 644 \
  --node-name k3s-master
```

Check:

```bash
kubectl get nodes
```

---

## BƯỚC 5: MỞ FIREWALL (UFW)

### Master:

```bash
sudo ufw allow 6443/tcp   # worker kết nối về master
sudo ufw allow 8472/udp   # pod giao tiếp
sudo ufw allow 10250/tcp  # lấy log pod
```

### Worker (bằng Ansible):

```bash
sudo ufw allow 8472/udp
sudo ufw allow 10250/tcp
```

---

## CÀI ANSIBLE TRÊN MASTER

```bash
sudo apt update
sudo apt install ansible -y
```

Test kết nối:

```bash
ansible all -i ~/k3s-inventory/hosts.ini -m ping
```

---

## SET SUDO KHÔNG PASSWORD (CHO WORKER)

Tạo file:

```bash
nano ~/k3s-inventory/setup-sudo.yml
```

```yaml
- hosts: workers
  become: yes
  tasks:
    - name: Allow thang2k6adu sudo without password
      copy:
        dest: /etc/sudoers.d/thang2k6adu
        content: |
          thang2k6adu ALL=(ALL) NOPASSWD:ALL
        owner: root
        group: root
        mode: '0440'
```

Run:

```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/setup-sudo.yml -K
```

---

## SET IP TĨNH CHO WORKER (OPTIONAL)

```bash
nano ~/k3s-inventory/set-static-ip.yml
```

```yaml
- hosts: workers
  become: yes
  vars:
    dns:
      - 8.8.8.8
      - 1.1.1.1

  tasks:
    - name: Disable cloud-init network
      copy:
        dest: /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
        content: |
          network: {config: disabled}

    - name: Remove old netplan config
      file:
        path: /etc/netplan/50-cloud-init.yaml
        state: absent

    - name: Configure static IP
      copy:
        dest: /etc/netplan/01-static.yaml
        content: |
          network:
            version: 2
            renderer: networkd
            ethernets:
              {{ ansible_default_ipv4.interface }}:
                dhcp4: no
                addresses:
                  - {{ hostvars[inventory_hostname].worker_ip }}/24
                gateway4: {{ ansible_default_ipv4.gateway }}
                nameservers:
                  addresses:
                    {% for d in dns %}
                    - {{ d }}
                    {% endfor %}

    - name: Apply netplan
      command: netplan apply
```

Run:

```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/set-static-ip.yml
```

Check:

```bash
ansible workers -i ~/k3s-inventory/hosts.ini -m shell -a \
"echo '=== HOST:' \$(hostname) && ip a | grep inet && ip route | grep default && ping -c 2 8.8.8.8"
```

---

## LẤY TOKEN TỪ MASTER

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Ví dụ:

```
K10a3f9c8c7b2a3b7f9::server:xxxxxxxx
```

---

## MỞ FIREWALL CHO WORKER (ANSIBLE)

```bash
nano ~/k3s-inventory/open-ufw-worker.yml
```

```yaml
- hosts: workers
  become: yes
  tasks:
    - name: Allow flannel VXLAN (8472/udp)
      ufw:
        rule: allow
        port: 8472
        proto: udp

    - name: Allow kubelet API (10250/tcp)
      ufw:
        rule: allow
        port: 10250
        proto: tcp

    - name: Enable UFW
      ufw:
        state: enabled
```

Run:

```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/open-ufw-worker.yml
```

---

## CÀI K3S AGENT (WORKER)

```bash
nano ~/k3s-inventory/install-k3s-worker.yml
```

```yaml
- hosts: workers
  become: yes
  vars:
    k3s_url: "https://192.168.0.104:6443"
    k3s_token: "K1028f433ac447753d1b7936476a06b90846c8d7d8ff9d3b54232c88638075cdb87::server:ed9d1a2aa018911c52cf9f100e36a157"

  tasks:
    - name: Install k3s agent
      shell: |
        curl -sfL https://get.k3s.io | K3S_URL={{ k3s_url }} K3S_TOKEN={{ k3s_token }} sh -
```

Run:

```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/install-k3s-worker.yml
```

---

## CHECK NODE ĐÃ JOIN

```bash
kubectl get nodes -o wide
```

Output:

```
NAME         STATUS   ROLES           IP
k3s-master   Ready    control-plane   192.168.0.104
worker1      Ready    <none>           192.168.0.105
worker2      Ready    <none>           192.168.0.106
```

---

## SET ROLE CHO WORKER

```bash
kubectl get nodes --no-headers | awk '{print $1}' | grep -v master | xargs -I {} kubectl label node {} node-role.kubernetes.io/worker=worker
```

Check:

```bash
kubectl get nodes
```

Output:

```
NAME            STATUS   ROLES    AGE
192.168.0.105   Ready    worker   1d
192.168.0.106   Ready    worker   1d
```

# 🚀 CÀI HELM + KUBERNETES DASHBOARD

## 1️⃣ Cài Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 2️⃣ Cài Kubernetes Dashboard

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Check:

```bash
kubectl get pods -n kubernetes-dashboard
```

---

## 3️⃣ Tạo ServiceAccount (tài khoản cho service)

Tạo file:

```bash
nano ~/k3s-inventory/dashboard-admin.yaml
```

Nội dung:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard
```

Apply:

```bash
kubectl apply -f ~/k3s-inventory/dashboard-admin.yaml
```

Check service:

```bash
kubectl get svc -n kubernetes-dashboard
```

---

## 4️⃣ Mở proxy để truy cập Dashboard

```bash
kubectl proxy --address=0.0.0.0 --accept-hosts='^.*$'
```

Nếu không mở proxy tại port `8001` thì phải vào `6443` (chắc chắn không vào được).

Truy cập Dashboard:

```
http://192.168.0.104:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Giải thích:

> “API Server, hãy forward request này tới Service kubernetes-dashboard, port tên là https (443), nó là port”

---

## 5️⃣ Lấy token để login Dashboard

```bash
kubectl -n kubernetes-dashboard create token kubernetes-dashboard-admin
```

---

## 6️⃣ Nếu SSH thì tạm mở port 8001

```bash
sudo ufw allow 8001
sudo ufw reload
```

Sau khi dùng xong thì đóng lại:

```bash
sudo ufw delete allow 8001
sudo ufw reload
```
Tất cả pod ở node nào?
kubectl get pods -A -o wide

---


test deploy nginx + node port

kubectl create namespace test-nginx

<!-- Lệnh này tạo deployment trên node bất kì (schedule tự chọn tối ưu) -->
kubectl create deployment nginx \
  --image=nginx \
  -n test-nginx

check
kubectl get pods -n test-nginx


expose

<!-- Này giống tạo 1 service port 80, node port bất kì trỏ về nginx
nó sẽ mở port của tất cả các node

yaml phải type node port, ko là nó về ClusterIP
 -->
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80 \
  -n test-nginx

check
kubectl get svc -n test-nginx

nginx   NodePort   10.43.7.190   <none>        80:30582/TCP   11s

vào
http://192.168.0.105:30582

scale thử

kubectl scale deployment -n test-nginx nginx --replicas=3
kubectl get pods -n test-nginx -o wide


rollback
kubectl delete namespace test-nginx

setup ingress (ko cần nodeport nữa)

ghét traefik nên disable đi

sudo nano /etc/rancher/k3s/config.yaml

disable:
  - traefik

sudo systemctl restart k3s

check
kubectl get pods -n kube-system

cài nginx

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

kubectl create namespace ingress-nginx

cái này cho reverse proxy, còn cloud có LB sẵn nên là khác

mkdir -p ~/k3s-inventory/nginx-ingress-config
nano ~/k3s-inventory/nginx-ingress-config/values.yaml

controller:
  replicaCount: 2

  ingressClassResource:
    enabled: true
    default: true
    name: nginx

  kind: Deployment

  service:
    enabled: true
    type: NodePort
    externalTrafficPolicy: Local
    ports:
      http: 80
      https: 443
    nodePorts:
      http: 30080
      https: 30443

  resources:
    requests:
      cpu: 200m
      memory: 256Mi

  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 60

  config:
    use-forwarded-headers: "true"
    proxy-real-ip-cidr: "0.0.0.0/0"
    real-ip-header: "X-Forwarded-For"
    proxy-body-size: "50m"
    proxy-read-timeout: "600"
    proxy-send-timeout: "600"
    worker-shutdown-timeout: "240s"
    enable-underscores-in-headers: "true"

  allowSnippetAnnotations: false

  metrics:
    enabled: true
    service:
      enabled: true
    serviceMonitor:
      enabled: true

  podDisruptionBudget:
    enabled: true
    minAvailable: 1

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/component
                operator: In
                values:
                  - controller
          topologyKey: kubernetes.io/hostname

  terminationGracePeriodSeconds: 300

  lifecycle:
    preStop:
      exec:
        command:
          - /wait-shutdown

defaultBackend:
  enabled: true


helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  -f ~/k3s-inventory/nginx-ingress-config/values.yaml

check
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

làm lại như cũ, khác là service lúc này là Cluster IP chứ ko dùng node port

kubectl create namespace test-nginx

kubectl create deployment nginx \
  --image=nginx \
  -n test-nginx

khác nè (không ghi type thì là ClusterIP), ko name thì cùng tên với deployment
ko định nghĩa target port thì tự lấy trong deployment

kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  -n test-nginx

kubectl get svc -n test-nginx

mkdir ~/k8s-manifest
nano ~/k8s-manifest/nginx-ingress.yaml

prefix sẽ match với tất cả

http://nginx.local/
http://nginx.local/abc
http://nginx.local/api
http://nginx.local/test/123

đều vào nginx hết

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: test-nginx
spec:
  rules:
  - host: nginx.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80

kubectl apply -f ~/k8s-manifest/nginx-ingress.yaml

check

kubectl get ingress -n test-nginx

map domain vào dns ở host

Ví dụ window, còn linux khá dễ thôi

chạy power shell bằng admin

notepad C:\Windows\System32\drivers\etc\hosts

flush dns (xóa cache)
ipconfig /flushdns

ping thử phát
ping nginx.local

sudo ufw allow 80
sudo ufw allow 443



vào
http://nginx.local

