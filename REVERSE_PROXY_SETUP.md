CLIENT (trình duyệt)
   |
   |  http://domain (80)  hoặc  https://domain (443)
   v
[VPS Public IP + Nginx + Certbot]
   |
   |  proxy_pass qua WireGuard VPN
   v
[WireGuard tunnel 10.10.10.0/24]
   |
   v
[NodePort Ingress NGINX trên các node K3s]
   |   (30080 cho HTTP, 30443 cho HTTPS)
   |
   v
[Ingress Controller]
   |
   v
[Service trong cluster]
   |
   v
[Pod (app, dashboard, v.v.)]


trên master

nano ~/k3s-inventory/install-wireguard.yml

- name: Install WireGuard on all servers
  hosts: all
  become: true
  tasks:
    - name: Update apt
      apt:
        update_cache: yes

    - name: Install wireguard
      apt:
        name: wireguard
        state: present
        
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/install-wireguard.yml

tạo key cho các máy

nano ~/k3s-inventory/gen-keys.yml

- name: Generate WireGuard keys
  hosts: all
  become: true
  tasks:
    - name: Create wireguard dir
      file:
        path: /etc/wireguard
        state: directory
        mode: 0700

    - name: Generate private key
      shell: wg genkey > /etc/wireguard/privatekey
      args:
        creates: /etc/wireguard/privatekey

    - name: Generate public key
      shell: cat /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
      args:
        creates: /etc/wireguard/publickey


chạy 

ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/gen-keys.yml


lấy public key của các máy

ansible -i ~/k3s-inventory/hosts.ini all -b -m shell -a "cat /etc/wireguard/publickey"

cat ~/k3s-inventory/hosts.ini

cài wireguard trên vps

sudo apt update
sudo apt install wireguard -y

tạo key
sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey'

lấy public key và lưu vào

sudo cat /etc/wireguard/publickey

vd JKL1bfmnfZfoS/QyQKIVW5mgENgNh4CyhlYi2ObqVUs=

giờ vps và các node đều có pub và private key

mục tiêu

VPS:
/etc/wireguard/wg0.conf  (chứa peer của tất cả node)


Mỗi node:
/etc/wireguard/wg0.conf  (kết nối về VPS)

sửa hosts.ini

MASTER=$(awk '/^\[master\]/{getline; print $1}' ~/k3s-inventory/hosts.ini)

ansible -i ~/k3s-inventory/hosts.ini master:workers -b -m shell -a "cat /etc/wireguard/publickey" --one-line \
| awk -v master="$MASTER" '
BEGIN{
  vpn=11
  print "[master]"
}
{
  ip=$1
  key=$NF
  if(ip==master){
    printf "%s ansible_user=thang2k6adu ansible_port=8022 worker_ip=%s vpn_ip=10.10.10.%d wg_public_key=%s\n",ip,ip,vpn,key
    vpn++
  }
}
END{
  print "\n[workers]"
}
' > ~/k3s-inventory/hosts.tmp.ini

ansible -i ~/k3s-inventory/hosts.ini master:workers -b -m shell -a "cat /etc/wireguard/publickey" --one-line \
| awk -v master="$MASTER" '
BEGIN{ vpn=12 }
{
  ip=$1
  key=$NF
  if(ip!=master){
    printf "%s ansible_user=thang2k6adu ansible_port=8022 worker_ip=%s vpn_ip=10.10.10.%d wg_public_key=%s\n",ip,ip,vpn,key
    vpn++
  }
}' >> ~/k3s-inventory/hosts.tmp.ini

mv ~/k3s-inventory/hosts.tmp.ini ~/k3s-inventory/hosts.ini


check 
cat ~/k3s-inventory/hosts.ini

phải ra
[master]
192.168.0.10 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.10 vpn_ip=10.10.10.11 wg_public_key=ui9LQVSQZOfQH5DzE1f/DtzPd2S6MFbOVTXqjgMPG1A=

[workers]
192.168.0.106 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.106 vpn_ip=10.10.10.12 wg_public_key=o7sRKClHG6qLHF9+2UTj8gtBcwK9zHZ6PEMdawADtGE=
192.168.0.105 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.105 vpn_ip=10.10.10.13 wg_public_key=1Vd2nl5yookdxx2BEdDaMfmOHwpfY+IMSYyu7GdJ2FQ=

khai báo thông tin vps

sửa hosts.ini

nano ~/k3s-inventory/hosts.ini

thêm (nhớ thay ip public, user và public key)
[vps]
13.229.60.179 ansible_user=ubuntu vpn_ip=10.10.10.1 wg_public_key=JKL1bfmnfZfoS/QyQKIVW5mgENgNh4CyhlYi2ObqVUs=

tạo wg config 0 trên các node

nano ~/k3s-inventory/gen-node-wg.yml

- name: Generate wg0.conf for nodes
  hosts: master:workers
  become: true
  vars:
    vps_ip: "{{ hostvars[groups['vps'][0]].inventory_hostname }}"
    vps_pubkey: "{{ hostvars[groups['vps'][0]].wg_public_key }}"

  tasks:
    - name: Read private key
      shell: cat /etc/wireguard/privatekey
      register: node_priv

    - name: Create wg0.conf
      copy:
        dest: /etc/wireguard/wg0.conf
        mode: 0600
        content: |
          [Interface]
          Address = {{ vpn_ip }}/24
          PrivateKey = {{ node_priv.stdout }}

          [Peer]
          PublicKey = {{ vps_pubkey }}
          Endpoint = {{ vps_ip }}:51820
          AllowedIPs = 10.10.10.0/24
          PersistentKeepalive = 25

run 
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/gen-node-wg.yml

check thử 1 node (master)
sudo cat /etc/wireguard/wg0.conf

phải ra
[Interface]
Address = 10.10.10.11/24
PrivateKey = 2MpNSUhR6Hb5VOPmwJPR4IE3M0FxB3Ib1QEARoJNnHY=

[Peer]
PublicKey = JKL1bfmnfZfoS/QyQKIVW5mgENgNh4CyhlYi2ObqVUs=
Endpoint = 13.229.60.179:51820
AllowedIPs = 10.10.10.0/24
PersistentKeepalive = 25

GEN file wg0 cho vps

nano ~/k3s-inventory/gen-vps-wg.yml

- name: Generate VPS wg0.conf
  hosts: localhost
  vars:
    vps_private_key: "PASTE_PRIVATE_KEY_VPS_HERE"

  tasks:
    - name: Build VPS config
      copy:
        dest: /home/thang2k6adu/k3s-inventory/wg0.vps.conf
        content: |
          [Interface]
          Address = 10.10.10.1/24
          ListenPort = 51820
          PrivateKey = {{ vps_private_key }}

          {% for host in groups['master'] + groups['workers'] %}
          [Peer]
          PublicKey = {{ hostvars[host].wg_public_key }}
          AllowedIPs = {{ hostvars[host].vpn_ip }}/32

          {% endfor %}

ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/gen-vps-wg.yml


check

 cat ~/k3s-inventory/wg0.vps.conf

lên vps

sudo cat /etc/wireguard/privatekey

sudo nano /etc/wireguard/wg0.conf

[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = PASTE_PRIVATE_KEY_VPS_HERE

[Peer]
PublicKey = ui9LQVSQZOfQH5DzE1f/DtzPd2S6MFbOVTXqjgMPG1A=
AllowedIPs = 10.10.10.11/32

[Peer]
PublicKey = o7sRKClHG6qLHF9+2UTj8gtBcwK9zHZ6PEMdawADtGE=
AllowedIPs = 10.10.10.12/32

[Peer]
PublicKey = 1Vd2nl5yookdxx2BEdDaMfmOHwpfY+IMSYyu7GdJ2FQ=
AllowedIPs = 10.10.10.13/32


trên vps bật ip forward cho vpn
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

mở port (EC2 thì lên trang) (51820 UDP)

chạy

sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0

check

sudo wg
ip a show wg0


trên LAN master

nano ~/k3s-inventory/start-wireguard.yml

- name: Start WireGuard on all hosts
  hosts: master:workers
  become: true
  tasks:
    - name: Enable wg-quick@wg0
      systemd:
        name: wg-quick@wg0
        enabled: yes

    - name: Start wg-quick@wg0
      systemd:
        name: wg-quick@wg0
        state: started

    - name: Show WireGuard status
      command: wg
      register: wg_status

    - name: Show wg0 interface
      command: ip a show wg0
      register: ip_status

    - name: Print wg status
      debug:
        msg: "{{ wg_status.stdout }}"

    - name: Print ip status
      debug:
        msg: "{{ ip_status.stdout }}"


chạy
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/start-wireguard.yml

test trên node (nào cũng được), nếu fail hãy mở port vps
ping 10.10.10.1

trên vps

cài nginx
sudo apt update
sudo apt install nginx -y


Lấy ip của tất cả các node (workers:master)

cài jq
sudo apt update
sudo apt install -y jq

lấy ip vpn
ansible-inventory -i ~/k3s-inventory/hosts.ini --list \
| jq -r '
._meta.hostvars
| to_entries[]
| select(.value.ansible_user=="thang2k6adu")
| "    server \(.value.vpn_ip):30080;"
'
echo "}
"

phải ra

upstream ingress_http {
    server 10.10.10.11:30080;
    server 10.10.10.13:30080;
    server 10.10.10.12:30080;
}

cấu hình nginx
sudo nano /etc/nginx/conf.d/k3s-ingress.conf

upstream ingress_http {
    least_conn;
    server 10.10.10.11:30080;
    server 10.10.10.13:30080;
    server 10.10.10.12:30080;
}

server {
    listen 80;

    location / {
        proxy_pass http://ingress_http;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

test và reload
sudo nginx -t
sudo systemctl reload nginx


test bên ngoài
http://13.229.60.179

phải ra welcome to nginx

lắp domain

lấy domain trỏ về vps (tự tìm hiểu)

rồi sửa lại

sudo nano /etc/nginx/conf.d/k3s-ingress.conf

upstream ingress_http {
    least_conn;
    server 10.10.10.11:30080;
    server 10.10.10.12:30080;
    server 10.10.10.13:30080;
}

server {
    listen 80;
    server_name thang2k6adu.xyz www.thang2k6adu.xyz;

    location / {
        proxy_pass http://ingress_http;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}

test + reload
sudo nginx -t
sudo systemctl reload nginx

nhớ mở 80 443 trên vps

test
http://thang2k6adu.xyz

bật https

cài certbot

sudo apt update
sudo apt install certbot python3-certbot-nginx -y

sudo certbot --nginx -d thang2k6adu.xyz -d www.thang2k6adu.xyz

chọn email

chọn
redirect HTTP → HTTPS = YES

CertBot tự sửa config thành

listen 443 ssl;
ssl_certificate /etc/letsencrypt/live/thang2k6adu.com/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/thang2k6adu.com/privkey.pem;

giờ sửa lại (trên master)
ansible-inventory -i ~/k3s-inventory/hosts.ini --list \
| jq -r '
._meta.hostvars
| to_entries[]
| select(.value.ansible_user=="thang2k6adu")
| "    server \(.value.vpn_ip):30443;"
'
echo "}
"

phải ra
    server 10.10.10.11:30443;
    server 10.10.10.13:30443;
    server 10.10.10.12:30443;

lên vps sửa lại

sudo nano /etc/nginx/conf.d/k3s-ingress.conf

nhớ sửa cả
location / {
    proxy_pass https://ingress_http;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_ssl_server_name on;
}

phải là 
upstream ingress_http {
    least_conn;

    server 10.10.10.11:30443;
    server 10.10.10.12:30443;
    server 10.10.10.13:30443;
}

server {
    listen 443 ssl;
    server_name thang2k6adu.xyz www.thang2k6adu.xyz;

    ssl_certificate     /etc/letsencrypt/live/thang2k6adu.xyz/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/thang2k6adu.xyz/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        proxy_pass https://ingress_http;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_ssl_server_name on;
    }
}

server {
    listen 80;
    server_name thang2k6adu.xyz www.thang2k6adu.xyz;

    if ($host = thang2k6adu.xyz) {
        return 301 https://$host$request_uri;
    }

    if ($host = www.thang2k6adu.xyz) {
        return 301 https://$host$request_uri;
    }

    return 404;
}


test
sudo nginx -t
sudo systemctl reload nginx

test
https://thang2k6adu.xyz

check auto renew
sudo certbot renew --dry-run

tạo test ingress dashboard (master)
thêm subdomain dashboard.thang2k6adu.xyz

rồi lên vps chạy
lệnh script thêm domain
sudo nano /usr/local/bin/add-domain.sh

#!/bin/bash

DOMAIN=$1

if [ -z "$DOMAIN" ]; then
  echo "Usage: add-domain.sh domain.com"
  exit 1
fi

CONF="/etc/nginx/conf.d/$DOMAIN.conf"

cat > $CONF <<EOF
server {
    listen 80;
    server_name $DOMAIN;

    location / {
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

certbot --nginx -d $DOMAIN

echo "DONE: https://$DOMAIN"


sudo chmod +x /usr/local/bin/add-domain.sh

dùng

sudo add-domain.sh dashboard.thang2k6adu.xyz

 (xin ssl cert)
sudo certbot --nginx -d dashboard.thang2k6adu.xyz

check
sudo nano /etc/nginx/conf.d/dashboard.thang2k6adu.xyz.conf


check
sudo nano /etc/nginx/conf.d/k3s-ingress.conf


nano ~/k8s-manifest/dashboard-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kubernetes-dashboard
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
spec:
  ingressClassName: nginx
  rules:
  - host: thang2k6adu.xyz
    http:
      paths:
      - path: /dashboard(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: kubernetes-dashboard
            port:
              number: 443


apply
kubectl apply -f ~/k8s-manifest/dashboard-ingress.yaml

check
https://thang2k6adu.xyz/dashboard

Client HTTPS
 → VPS Nginx (443)
 → proxy_pass tới node:30443
 → Ingress NGINX (443)
 → Service kubernetes-dashboard:443
 → Pod dashboard
