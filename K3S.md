# 🚀 HƯỚNG DẪN TRIỂN KHAI K3S CLUSTER (MASTER + WORKER)

## BƯỚC 1: ĐỔI HOSTNAME (TRÊN NODE MASTER)

Nhớ dùng `ip a` để check **IP / mask / gateway** và thay cho đúng trước khi làm bất cứ điều gì.
```bash
sudo hostnamectl set-hostname k3s-master
sudo nano /etc/hosts
```

Ví dụ nội dung:
```txt
127.0.0.1 localhost
192.168.0.50 k3s-master
```

Reboot:
```bash
sudo reboot
```

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
        - 192.168.0.50/24
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

## BƯỚC 3: SCAN IP CÁC SERVER WORKER (TRÊN MASTER)

Cài `nmap`:
```bash
sudo apt install nmap -y
```

Auto generate inventory file

⚠️ Nhớ sửa subnet + port SSH cho đúng môi trường. Sau này thêm server thì nhớ chạy lại cái này là oke.
```bash
SUBNET=192.168.0.0/24
PORT=8022
USER="thang2k6adu"
MASTER_IP=$(hostname -I | awk '{print $1}')

mkdir -p ~/k3s-inventory && cd ~/k3s-inventory

echo -e "[master]\n$MASTER_IP ansible_user=$USER ansible_port=$PORT worker_ip=$MASTER_IP\n\n[workers]" > hosts.ini

sudo nmap -p $PORT --open $SUBNET \
| grep "Nmap scan report" \
| grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" \
| grep -v "^$MASTER_IP$" \
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
192.168.0.50 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.50

[workers]
192.168.0.505 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.505
192.168.0.506 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.506
```

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

## CÀI ANSIBLE TRÊN MASTER
```bash
sudo apt update
sudo apt install ansible -y
```

Lưu ý phải lắp ssh vào master node trước khi ssh

tạo ssh
ssh-keygen -t ed25519 -C "k3s-master"

lấy public key
cat ~/.ssh/id_ed25519.pub

phân quyền
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

cho vào các node
phân quyền
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys


```

Test kết nối:
```bash
ansible workers -i ~/k3s-inventory/hosts.ini -m ping
```

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
      async: 10
      poll: 0
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

## LẤY TOKEN TỪ MASTER
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Ví dụ:
```
K10a3f9c8c7b2a3b7f9::server:xxxxxxxx
```

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

## CÀI K3S AGENT (WORKER)
```bash
nano ~/k3s-inventory/install-k3s-worker.yml
```
```yaml
- hosts: workers
  become: yes
  vars:
    k3s_url: "https://192.168.0.50:6443"
    k3s_token: "K1003411f61437bb01f4e057e32c1c7ab85849b89c256e5402cd739a91bbafeb487::server:c1b0dc8341dd56fa744edee71ff297c0"

  tasks:
    - name: Install k3s agent
      shell: |
        curl -sfL https://get.k3s.io | K3S_URL={{ k3s_url }} K3S_TOKEN={{ k3s_token }} sh -
```

Run:
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/install-k3s-worker.yml
```

Uninstall nếu lỗi:
```bash
nano ~/k3s-inventory/uninstall-k3s-worker.yml
```
```yaml
- hosts: workers
  become: yes

  tasks:
    - name: Stop k3s-agent service
      systemd:
        name: k3s-agent
        state: stopped
        enabled: false
      ignore_errors: yes

    - name: Run k3s-agent uninstall script
      shell: |
        if [ -f /usr/local/bin/k3s-agent-uninstall.sh ]; then
          /usr/local/bin/k3s-agent-uninstall.sh
        fi
      args:
        warn: false
      ignore_errors: yes

    - name: Remove k3s directories
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /etc/rancher/k3s
        - /var/lib/rancher/k3s
        - /var/lib/kubelet
      ignore_errors: yes
```
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/uninstall-k3s-worker.yml
```

## CHECK NODE ĐÃ JOIN
```bash
kubectl get nodes -o wide
```

Output:
```
NAME         STATUS   ROLES           IP
k3s-master   Ready    control-plane   192.168.0.50
worker1      Ready    <none>          192.168.0.505
worker2      Ready    <none>          192.168.0.506
```

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
192.168.0.505   Ready    worker   1d
192.168.0.506   Ready    worker   1d
```

# 🚀 CÀI HELM + KUBERNETES DASHBOARD

## 1️⃣ Cài Helm
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

## 2️⃣ Cài Kubernetes Dashboard
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

Check:
```bash
kubectl get pods -n kubernetes-dashboard
```

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

## 4️⃣ Mở proxy để truy cập Dashboard
```bash
sudo ufw allow 8001
kubectl proxy --address=0.0.0.0 --accept-hosts='^.*$'
```

Nếu không mở proxy tại port `8001` thì phải vào `6443` (chắc chắn không vào được).

Truy cập Dashboard:
```
http://192.168.0.50:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Giải thích:

> "API Server, hãy forward request này tới Service kubernetes-dashboard, port tên là https (443), nó là port"

## 5️⃣ Lấy token để login Dashboard
```bash
kubectl -n kubernetes-dashboard create token kubernetes-dashboard-admin
```

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
```bash
kubectl get pods -A -o wide
```

## TEST DEPLOY NGINX + NODE PORT
```bash
kubectl create namespace test-nginx
```

Lệnh này tạo deployment trên node bất kì (schedule tự chọn tối ưu):
```bash
kubectl create deployment nginx \
  --image=nginx \
  -n test-nginx
```

Check:
```bash
kubectl get pods -n test-nginx
```

### Expose

Này giống tạo 1 service port 80, node port bất kì trỏ về nginx. Nó sẽ mở port của tất cả các node.

YAML phải type node port, không là nó về ClusterIP:
```bash
kubectl expose deployment nginx \
  --type=NodePort \
  --port=80 \
  -n test-nginx
```

Check:
```bash
kubectl get svc -n test-nginx
```

Output:
```
nginx   NodePort   10.43.7.190   <none>        80:30582/TCP   11s
```

Vào:
```
http://192.168.0.505:30582
```

### Scale thử
```bash
kubectl scale deployment -n test-nginx nginx --replicas=3
kubectl get pods -n test-nginx -o wide
```

### Rollback
```bash
kubectl delete namespace test-nginx
```

## SETUP INGRESS (KHÔNG CẦN NODEPORT NỮA)

Ghét traefik nên disable đi:
```bash
sudo nano /etc/rancher/k3s/config.yaml
```

Nội dung:
```yaml
disable:
  - traefik
```
```bash
sudo systemctl restart k3s
```

Check:
```bash
kubectl get pods -n kube-system
```

Config kube:
```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

fix lỗi 127.0.0.1
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
source ~/.bashrc

### Trước khi cài nginx, cài monitoring
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring
```

Check:
```bash
kubectl get pods -n monitoring
```

Output:
```
prometheus-...
grafana-...
alertmanager-...
node-exporter-...
```

### Cài nginx
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Cái này cho reverse proxy, còn cloud có LB sẵn nên là khác:
```bash
mkdir -p ~/k3s-inventory/nginx-ingress-config
nano ~/k3s-inventory/nginx-ingress-config/values.yaml
```

Nội dung:
```yaml
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
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
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
```

Install:
```bash
kubectl create namespace ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx \
  -f ~/k3s-inventory/nginx-ingress-config/values.yaml
```

Nếu lỗi:
```bash
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```

Check:
```bash
kubectl get pods -n ingress-nginx -o wide
kubectl get svc -n ingress-nginx
```

### Làm lại như cũ, khác là service lúc này là Cluster IP chứ không dùng node port
```bash
kubectl create namespace test-nginx

kubectl create deployment nginx \
  --image=nginx \
  -n test-nginx
```

Khác nè (không ghi type thì là ClusterIP), không name thì cùng tên với deployment. Không định nghĩa target port thì tự lấy trong deployment:
```bash
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80 \
  -n test-nginx
```
```bash
kubectl get svc -n test-nginx
```
```bash
mkdir ~/k8s-manifest
nano ~/k8s-manifest/nginx-ingress.yaml
```

Prefix sẽ match với tất cả:
```
http://nginx.local/
http://nginx.local/abc
http://nginx.local/api
http://nginx.local/test/123
```

Đều vào nginx hết:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-ingress
  namespace: test-nginx
spec:
  rules:
  - host: kruzetech.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```
```bash
kubectl apply -f ~/k8s-manifest/nginx-ingress.yaml
```

Check:
```bash
kubectl get ingress -n test-nginx
```

### Map domain vào DNS ở host

Ví dụ Windows, còn Linux khá dễ thôi.

Chạy PowerShell bằng admin:
```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```

Thêm vào:
```
192.168.0.505 nginx.local
```

(Lưu ý là chỉ node nào có pod mới được)

Flush DNS (xóa cache):
```powershell
ipconfig /flushdns
```

Ping thử phát:
```powershell
ping nginx.local
```
```bash
sudo ufw allow 80
sudo ufw allow 443
```

Vào:
```
http://nginx.local
```

