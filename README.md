# 🧱 PXE BOOT + AUTOINSTALL UBUNTU SERVER

## 🎯 HỖ TRỢ CẢ BIOS & UEFI (CHUNG 1 HỆ THỐNG)

Before do this, remember to check your network with `ip a`, mask, gateway and replace all of them in my docs., nhớ set ip tĩnh sau khi cài server

> Áp dụng cho:
>
> * Bare metal **CHƯA CÓ OS**
> * PXE boot **BIOS hoặc UEFI**
> * Ubuntu Server **22.04+**
> * Secure Boot **TẮT**
> * Mục tiêu: **Zero-touch autoinstall**

---

## 0️⃣ TIỀN ĐỀ (BẮT BUỘC CHECK)

Trên PXE server:

```bash
ip a
```

Xác nhận:

* NIC: `ens33`
* IP: `192.168.0.103/24`

⚠️ **Nếu NIC khác → phải sửa TOÀN BỘ config theo NIC đó**

---

## I️⃣ TƯ DUY CHUẨN (RẤT QUAN TRỌNG)

### 🔹 BIOS PXE

```
BIOS
 └─ PXE
     └─ DHCP → pxelinux.0
         └─ pxelinux.cfg/default
             └─ vmlinuz + initrd
                 └─ Installer
```

### 🔹 UEFI PXE

```
UEFI
 └─ PXE
     └─ DHCP → grubx64.efi
         └─ grub.cfg
             └─ vmlinuz + initrd
                 └─ Installer
```

👉 **Khác nhau CHỈ ở bootloader**

* BIOS → **pxelinux**
* UEFI → **GRUB EFI**

👉 **Kernel cmdline giống nhau 100%**

---

## II️⃣ PHASE 1 — DHCP + PXE (dnsmasq)

### 1️⃣ Cài dnsmasq

```bash
sudo apt update
sudo apt install -y dnsmasq
```

---

### 2️⃣ Cấu hình DHCP + PXE (BIOS + UEFI)

```bash
sudo nano /etc/dnsmasq.d/pxe.conf
```

```conf
interface=ens33
bind-interfaces

# DHCP range
dhcp-range=192.168.0.50,192.168.0.100,12h
dhcp-authoritative

# Gateway + DNS
dhcp-option=3,192.168.0.1
dhcp-option=6,192.168.0.103

# Domain nội bộ
dhcp-option=15,lab.local
domain=lab.local
expand-hosts

# TFTP
enable-tftp
tftp-root=/srv/tftp

# Detect firmware
dhcp-match=set:bios,option:client-arch,0
dhcp-match=set:uefi,option:client-arch,7

# Bootloader
dhcp-boot=tag:bios,pxelinux.0
dhcp-boot=tag:uefi,grubx64.efi
```

Test & restart:

```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
journalctl -u dnsmasq -f
```

đến đây có thể bị lỗi do /srv/tftp chưa tạo nên dnsmasq lỗi là bình thườn
típ nhe

---

## III️⃣ PHASE 2 — TFTP BOOTLOADER

### 🔹 BIOS: pxelinux

```bash
sudo apt install -y pxelinux syslinux-common
sudo mkdir -p /srv/tftp/pxelinux.cfg
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/
```

---

### 🔹 UEFI: GRUB EFI

```bash
sudo apt install -y grub-efi-amd64-bin
sudo mkdir -p /srv/tftp/grub
sudo cp /usr/lib/grub/x86_64-efi/monolithic/grubx64.efi /srv/tftp/
```

```bash
sudo chown -R nobody:nogroup /srv/tftp
sudo chmod -R 755 /srv/tftp
```
check 
```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
sudo systemctl status dnsmasq
journalctl -u dnsmasq -f
```

---

### 🔍 Test TFTP

(OPEN port below this, end of this file before do this)

```bash
sudo ufw allow 67/udp
sudo ufw allow 68/udp
sudo ufw allow 69/udp
sudo ufw allow 80/tcp
sudo ufw allow from 192.168.0.0/24
```

```bash
sudo apt install tftp
tftp 192.168.0.103
get grubx64.efi
tftp 192.168.0.103
get pxelinux.0
```

---

## IV️⃣ PHASE 3 — KERNEL + INITRD

### 3️⃣ Chuẩn bị ISO qua HTTP (TỰ TẢI) NFS ROOT FILESYSTEM

```bash
sudo mkdir -p /srv/http/ubuntu
cd /srv/http/ubuntu
sudo wget https://releases.ubuntu.com/22.04/ubuntu-22.04.5-live-server-amd64.iso

sudo apt update
sudo apt install -y nfs-kernel-server

```

Mount:

```bash
sudo mkdir -p /srv/nfs/ubuntu
sudo mount -o loop /srv/http/ubuntu/ubuntu-22.04.5-live-server-amd64.iso /mnt
sudo cp -a /mnt/. /srv/nfs/ubuntu/
sudo umount /mnt
```

### 3️⃣ Copy kernel + initrd

```bash
sudo mkdir -p /srv/tftp/ubuntu
sudo cp /srv/nfs/ubuntu/casper/vmlinuz /srv/tftp/ubuntu/
sudo cp /srv/nfs/ubuntu/casper/initrd /srv/tftp/ubuntu/
```

```bash
sudo nano /etc/exports
```

```conf
/srv/nfs/ubuntu *(ro,sync,no_subtree_check)
```

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

---

## V️⃣ PHASE 4 — BOOT MENU (KERNEL CMDLINE = NÃO)

### 🔹 BIOS — pxelinux

(config **ko được xuống dòng**)

```bash
sudo nano /srv/tftp/pxelinux.cfg/default
```

```cfg
DEFAULT install
PROMPT 0
TIMEOUT 30

LABEL install
  KERNEL ubuntu/vmlinuz
  INITRD ubuntu/initrd
  APPEND ip=dhcp boot=casper netboot=nfs nfsroot=192.168.0.103:/srv/nfs/ubuntu autoinstall ignore_uuid fsck.mode=skip ds=nocloud-net;s=http://192.168.0.103/autoinstall/ ---
```

---

### 🔹 UEFI — GRUB

```bash
sudo nano /srv/tftp/grub/grub.cfg
```

```cfg
set timeout=30
set default=0

menuentry "Install Ubuntu Server (NFS Boot - Low RAM)" {
    linux /ubuntu/vmlinuz ip=dhcp boot=casper netboot=nfs nfsroot=192.168.0.103:/srv/nfs/ubuntu autoinstall ignore_uuid fsck.mode=skip ds=nocloud-net\;s=http://192.168.0.103/autoinstall/
    initrd /ubuntu/initrd
}
```

---

## VI️⃣ PHASE 5 — HTTP SERVER

```bash
sudo apt install -y nginx
sudo rm /etc/nginx/sites-enabled/default
```

```bash
sudo nano /etc/nginx/sites-available/pxe
```

```nginx
server {
    listen 80;
    root /srv/http;
    autoindex on;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/pxe /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

Test:

```bash
curl http://192.168.0.103/ubuntu/
```

---

## VII️⃣ PHASE 6 — AUTOINSTALL (cloud-init)

```bash
sudo mkdir -p /srv/http/autoinstall
```

### meta-data

```bash
sudo nano /srv/http/autoinstall/meta-data
```

```yaml
```

### user-data

```bash
sudo nano /srv/http/autoinstall/user-data
```

<!-- Nhớ copy cả #cloud-config -->
```yaml
#cloud-config
autoinstall:
  version: 1

  # disable interact
  interactive-sections: []

  # auto reboot when successfully install
  shutdown: reboot

  locale: en_US.UTF-8
  keyboard:
    layout: us
    variant: ""

  identity:
    hostname: localhost
    username: thang2k6adu
    password: "$6$3KSEmEFffX6Gb5oH$4VL.PXtoT1bjs3UAwHmtaGRCByvzqn2PG3hoJ71.EeXC7KHdqSaOEN9No54uLcBPVSsOWptDc39WY3DmeftCi1"

  ssh:
    install-server: true
    allow-pw: false
    authorized-keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPrDNYZt+doMzqGwcElOycPaHKoMmZ9743pAVw9Q29KC thang2k6adu@gmail.com

  network:
    version: 2
    ethernets:
      default:
        match:
          name: "e*"
        dhcp4: true

  storage:
    layout:
      name: lvm

  timezone: Asia/Ho_Chi_Minh

  packages:
    - curl
    - wget
    - git
    - vim
    - htop
    - net-tools
    - ufw
    - fail2ban
    - chrony
    - ca-certificates

  late-commands:
    - |
      curtin in-target -- bash -c '
      MY_IP=$(hostname -I | awk "{print \$1}")
      MY_ID=$(echo $MY_IP | awk -F. "{print \$4}")
      NEW_HOSTNAME="node-$MY_ID"

      echo "--> Setup Hostname: $NEW_HOSTNAME (IP: $MY_IP)"

      echo "$NEW_HOSTNAME" > /etc/hostname
      hostnamectl set-hostname $NEW_HOSTNAME

      sed -i "/127.0.1.1/d" /etc/hosts
      sed -i "/127.0.0.1/a 127.0.1.1 $NEW_HOSTNAME" /etc/hosts
      '

    # Enable services
    - curtin in-target -- systemctl enable ssh.service
    - curtin in-target -- systemctl enable ufw
    - curtin in-target -- systemctl enable fail2ban
    - curtin in-target -- systemctl enable chrony

    # Firewall basic rules
    - curtin in-target -- ufw default deny incoming
    - curtin in-target -- ufw default allow outgoing
    - curtin in-target -- ufw allow 8022/tcp
    - curtin in-target -- ufw --force enable

    # SSH hardening
    - curtin in-target -- sed -i 's/^#\?Port .*/Port 8022/' /etc/ssh/sshd_config
    - curtin in-target -- sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config
    - curtin in-target -- sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
    - curtin in-target -- sed -i 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' /etc/ssh/sshd_config
    - curtin in-target -- sed -i 's/^#MaxAuthTries.*/MaxAuthTries 3/' /etc/ssh/sshd_config
    - curtin in-target -- sed -i 's/^#ClientAliveInterval.*/ClientAliveInterval 300/' /etc/ssh/sshd_config
    - curtin in-target -- sed -i 's/^#ClientAliveCountMax.*/ClientAliveCountMax 2/' /etc/ssh/sshd_config

    - curtin in-target -- systemctl restart ssh.service

    # Fail2ban basic config
    - curtin in-target -- cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    - curtin in-target -- sed -i 's/^bantime.*/bantime = 3600/' /etc/fail2ban/jail.local
    - curtin in-target -- sed -i 's/^findtime.*/findtime = 600/' /etc/fail2ban/jail.local
    - curtin in-target -- sed -i 's/^maxretry.*/maxretry = 3/' /etc/fail2ban/jail.local
    - curtin in-target -- bash -c "printf '[sshd]\nenabled = true\nport = 8022\n' > /etc/fail2ban/jail.d/sshd.local"
    - curtin in-target -- systemctl restart fail2ban
    - curtin in-target -- cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
    - curtin in-target -- bash -c "printf '[sshd]\nenabled = true\nbackend = systemd\nport = 8022\nmaxretry = 3\nbantime = 3600\n' > /etc/fail2ban/jail.d/sshd.local"
    - curtin in-target -- systemctl restart fail2ban

    - curtin in-target -- swapoff -a
    - curtin in-target -- sed -i '/swap/d' /etc/fstab
    - curtin in-target -- bash -c "echo 'vm.swappiness=0' > /etc/sysctl.d/99-k8s.conf"
    - curtin in-target -- sysctl --system
```

---

## VIII️⃣ PHASE 7 — DHCP RESERVATION (KHUYẾN NGHỊ)

```conf
dhcp-host=AA:BB:CC:DD:EE:01,node-01,192.168.0.11
dhcp-host=AA:BB:CC:DD:EE:02,node-02,192.168.0.12
```

---

## IX️⃣ DEBUG NHANH (CHUẨN OPS)

```bash
journalctl -u dnsmasq -f
tcpdump -i ens33 port 67 or port 69
cat /proc/cmdline
```

👉 **Nếu không thấy `autoinstall` trong `/proc/cmdline` → nó SẼ HỎI**

---

## 🔒 TỔNG KẾT (NHỚ KỸ)

* BIOS ≠ UEFI → **chỉ khác bootloader**
* Kernel cmdline **PHẢI có `autoinstall`**
* PXE sạch → debug theo tầng
* Không có hack, không có shortcut

### ERR PXE-E51: No DHCP or proxyDHCP offers were received

Lỗi này là do config **dnsmasq**.
Xem kỹ `ip a`, đối chiếu với config file của dnsmasq (`pxe.conf`).

Các điểm cần kiểm tra:

* IP interface
* Gateway
* DNS
* DHCP range

Nếu dùng vmware máy ảo, nhớ bật hết bridge lên, nếu ko sẽ ko broadcast được.

### ERR PXE-EA0: Network boot canceled by keystroke

Timeout, tắt đi bật lại, chờ lâu.

### Can't open /dev/sr0: No medium found

Không mount được iso, fall back về `/dev`.

Giải pháp:

* `boot=casper` (đã thêm)
* Check URL ubuntu và autoinstall trong:

  * `/srv/tftp/grub/grub.cfg`
  * `/srv/tftp/pxelinux.cfg/default`
* Phải đúng với `ip a`
* Thêm:

  ```
  APPEND ip=dhcp rd.neednet=1
  ```

---

QUản lý node

sudo nano /etc/systemd/resolved.conf

[Resolve]
DNS=127.0.0.1
Domains=lab.local

sudo systemctl restart systemd-resolved
resolvectl status

ping node-109
ping node-109.lab.local
ping google.com

genisoimage -output cidata.iso -volid cidata -joliet -rock meta-data user-data