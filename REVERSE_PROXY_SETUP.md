# Hướng Dẫn Cài Đặt WireGuard VPN và Nginx Reverse Proxy cho K3s Cluster

## Sơ Đồ Kiến Trúc

```
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
```

---

## Phần 1: Cài Đặt WireGuard

### 1.1. Cài đặt WireGuard trên tất cả server (trên master)

**Tạo file playbook:**
```bash
nano ~/k3s-inventory/install-wireguard.yml
```

**Nội dung `install-wireguard.yml`:**
```yaml
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
```

**Chạy playbook:**
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/install-wireguard.yml
```

---

### 1.2. Tạo key cho các máy

**Tạo file playbook:**
```bash
nano ~/k3s-inventory/gen-keys.yml
```

**Nội dung `gen-keys.yml`:**
```yaml
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
```

**Chạy playbook:**
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/gen-keys.yml
```

**Lấy public key của các máy:**
```bash
ansible -i ~/k3s-inventory/hosts.ini all -b -m shell -a "cat /etc/wireguard/publickey"
```

**Check file hosts:**
```bash
cat ~/k3s-inventory/hosts.ini
```

---

### 1.3. Cài WireGuard trên VPS

**Cài đặt:**
```bash
sudo apt update
sudo apt install wireguard -y
```

**Tạo key:**
```bash
sudo sh -c 'umask 077; wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey'
```

**Lấy public key và lưu vào:**
```bash
sudo cat /etc/wireguard/publickey
```

Ví dụ: `JKL1bfmnfZfoS/QyQKIVW5mgENgNh4CyhlYi2ObqVUs=`

---

### 1.4. Mục tiêu cấu hình

**VPS:** (  (chứa peer của tất cả node))
```
/etc/wireguard/wg0.conf
```

**Mỗi node:**  (kết nối về VPS)
```
/etc/wireguard/wg0.conf 
```

---

### 1.5. Cập nhật hosts.ini với thông tin VPN

**Lấy IP master:**
```bash
MASTER=$(awk '/^\[master\]/{getline; print $1}' ~/k3s-inventory/hosts.ini)
```

**Tạo section [master]:**
```bash
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
```

**Tạo section [workers]:**
```bash
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
```

**Di chuyển file:**
```bash
mv ~/k3s-inventory/hosts.tmp.ini ~/k3s-inventory/hosts.ini
```

**Check kết quả:**
```bash
cat ~/k3s-inventory/hosts.ini
```

**Phải ra:**
```ini
[master]
192.168.0.10 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.10 vpn_ip=10.10.10.11 wg_public_key=ui9LQVSQZOfQH5DzE1f/DtzPd2S6MFbOVTXqjgMPG1A=

[workers]
192.168.0.106 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.106 vpn_ip=10.10.10.12 wg_public_key=o7sRKClHG6qLHF9+2UTj8gtBcwK9zHZ6PEMdawADtGE=
192.168.0.105 ansible_user=thang2k6adu ansible_port=8022 worker_ip=192.168.0.105 vpn_ip=10.10.10.13 wg_public_key=1Vd2nl5yookdxx2BEdDaMfmOHwpfY+IMSYyu7GdJ2FQ=
```

---

### 1.6. Khai báo thông tin VPS

**Sửa hosts.ini:**
```bash
nano ~/k3s-inventory/hosts.ini
```

**Thêm (nhớ thay ip public, user và public key):**
```ini
[vps]
13.229.60.179 ansible_user=ubuntu vpn_ip=10.10.10.1 wg_public_key=JKL1bfmnfZfoS/QyQKIVW5mgENgNh4CyhlYi2ObqVUs=
```

---

### 1.7. Tạo wg0.conf trên các node

**Tạo playbook:**
```bash
nano ~/k3s-inventory/gen-node-wg.yml
```

**Nội dung `gen-node-wg.yml`:**
```yaml
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
```

**Chạy playbook:**
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/gen-node-wg.yml
```

**Check thử 1 node (master):**
```bash
sudo cat /etc/wireguard/wg0.conf
```

**Phải ra:**
```ini
[Interface]
Address = 10.10.10.11/24
PrivateKey = 2MpNSUhR6Hb5VOPmwJPR4IE3M0FxB3Ib1QEARoJNnHY=

[Peer]
PublicKey = JKL1bfmnfZfoS/QyQKIVW5mgENgNh4CyhlYi2ObqVUs=
Endpoint = 13.229.60.179:51820
AllowedIPs = 10.10.10.0/24
PersistentKeepalive = 25
```

---

### 1.8. Gen file wg0 cho VPS

**Tạo playbook:**
```bash
nano ~/k3s-inventory/gen-vps-wg.yml
```

**Nội dung `gen-vps-wg.yml`:**
```yaml
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
```

**Chạy playbook:**
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/gen-vps-wg.yml
```

**Check:**
```bash
cat ~/k3s-inventory/wg0.vps.conf
```

---

### 1.9. Cấu hình wg0.conf trên VPS

**Lên VPS, lấy private key:**
```bash
sudo cat /etc/wireguard/privatekey
```

**Tạo file config:**
```bash
sudo nano /etc/wireguard/wg0.conf
```

**Nội dung:**
```ini
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
```

---

### 1.10. Bật IP forward trên VPS

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Mở port (EC2 thì lên trang) (51820 UDP)**

---

### 1.11. Khởi động WireGuard trên VPS

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo systemctl status wg-quick@wg0
```

**Check:**
```bash
sudo wg
ip a show wg0
```

---

### 1.12. Khởi động WireGuard trên các node (trên LAN master)

**Tạo playbook:**
```bash
nano ~/k3s-inventory/start-wireguard.yml
```

**Nội dung `start-wireguard.yml`:**
```yaml
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
```

**Chạy:**
```bash
ansible-playbook -i ~/k3s-inventory/hosts.ini ~/k3s-inventory/start-wireguard.yml
```

**Test trên node (nào cũng được), nếu fail hãy mở port vps:**
```bash
ping 10.10.10.1
```

---