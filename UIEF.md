
---

# 🧱 PXE BOOT + AUTOINSTALL — **UEFI ONLY (PURE EFI)**

> Áp dụng cho:
>
> * Bare metal **CHƯA OS**
> * **UEFI PXE**
> * **KHÔNG Legacy / BIOS**
> * **KHÔNG pxelinux**
> * Secure Boot **TẮT**
> * Mục tiêu: **cài Ubuntu Server tự động**

---

## 0️⃣ TIỀN ĐỀ (CHECK TRƯỚC)

Trên PXE server:

```bash
ip a
```

Xác nhận:

* NIC đúng: `ens33`
* IP đúng: `192.168.115.129/24`

❗ **Nếu NIC khác → sửa TOÀN BỘ config theo NIC đó**

---

## I️⃣ TƯ DUY CHUẨN (UEFI PXE)

UEFI PXE = **GRUB EFI**, **KHÔNG SYSLINUX**

Chuỗi boot **CHUẨN**:

```
UEFI
 └─ PXE
     └─ DHCP (IP + grubx64.efi)
         └─ TFTP
             └─ grubx64.efi
                 └─ grub.cfg
                     └─ vmlinuz + initrd
                         └─ Ubuntu Installer
```

👉 **pxelinux = 0%**
👉 **Bootloader = grubx64.efi**

---

## II️⃣ PHASE 1 — DHCP + PXE (UEFI)

### 1️⃣ Cài dnsmasq

```bash
sudo apt update
sudo apt install -y dnsmasq
```

Check:

```bash
systemctl status dnsmasq --no-pager
```

---

### 2️⃣ Cấu hình DHCP cho **UEFI PXE**

```bash
sudo nano /etc/dnsmasq.d/pxe-uefi.conf
```

**DÁN NGUYÊN KHỐI:**

```conf
interface=ens33
bind-interfaces

dhcp-range=192.168.115.50,192.168.115.100,12h
dhcp-authoritative

dhcp-option=3,192.168.115.1
dhcp-option=6,192.168.115.129

# Domain LAN
dhcp-option=15,lab.local
domain=lab.local
expand-hosts

# PXE UEFI
enable-tftp
tftp-root=/srv/tftp

# EFI x86_64
dhcp-match=set:efi64,option:client-arch,7
dhcp-boot=tag:efi64,grubx64.efi
```

Test & restart:

```bash
sudo dnsmasq --test
sudo systemctl restart dnsmasq
```

Log:

```bash
journalctl -u dnsmasq -f
```

---

## III️⃣ PHASE 2 — TFTP BOOTLOADER (GRUB EFI)

### 3️⃣ Cài GRUB EFI

```bash
sudo apt install -y grub-efi-amd64-bin
```

---

### 4️⃣ Tạo cấu trúc TFTP

```bash
sudo mkdir -p /srv/tftp/grub
```

---

### 5️⃣ Copy GRUB EFI binary

```bash
sudo cp /usr/lib/grub/x86_64-efi/grubx64.efi /srv/tftp/
```

Check:

```bash
ls -lh /srv/tftp
```

Phải thấy:

```
grubx64.efi
grub/
```

---

### 6️⃣ Test TFTP thủ công

```bash
tftp 192.168.115.129
tftp> get grubx64.efi
tftp> quit
```

👉 Get được = TFTP OK

---

## IV️⃣ PHASE 3 — KERNEL + INITRD

### 7️⃣ Chuẩn bị HTTP ISO

```bash
sudo mkdir -p /srv/http/ubuntu
sudo cp ubuntu-22.04.5-live-server-amd64.iso /srv/http/ubuntu/
```

Mount ISO:

```bash
sudo mkdir -p /mnt/iso
sudo mount -o loop /srv/http/ubuntu/ubuntu-22.04.5-live-server-amd64.iso /mnt/iso
```

Check:

```bash
ls /mnt/iso/casper
```

Phải có:

```
vmlinuz
initrd
```

---

### 8️⃣ Copy kernel + initrd

```bash
sudo mkdir -p /srv/tftp/ubuntu

sudo cp /mnt/iso/casper/vmlinuz /srv/tftp/ubuntu/
sudo cp /mnt/iso/casper/initrd /srv/tftp/ubuntu/
```

Unmount:

```bash
sudo umount /mnt/iso
```

---

## V️⃣ PHASE 4 — GRUB MENU (TRUNG TÂM ĐIỀU KHIỂN)

### 9️⃣ Tạo `grub.cfg`

```bash
sudo nano /srv/tftp/grub/grub.cfg
```

**DÁN NGUYÊN KHỐI:**

```cfg
set timeout=5
set default=0

menuentry "Install Ubuntu Server (UEFI PXE)" {
    linux /ubuntu/vmlinuz ip=dhcp \
        url=http://192.168.115.129/ubuntu/ubuntu-22.04.5-live-server-amd64.iso \
        autoinstall \
        ds=nocloud-net;s=http://192.168.115.129/autoinstall/ ---
    initrd /ubuntu/initrd
}
```

👉 **GRUB CMDLINE = não của installer**

---

## VI️⃣ PHASE 5 — HTTP SERVER

### 🔟 Cài nginx

```bash
sudo apt install -y nginx
```

---

### 1️⃣1️⃣ Xoá site mặc định

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

### 1️⃣2️⃣ Tạo site PXE

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

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/pxe /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

Test:

```bash
curl http://192.168.115.129/ubuntu/
```

---

## VII️⃣ PHASE 6 — AUTOINSTALL (CLOUD-INIT)

### 1️⃣3️⃣ Thư mục autoinstall

```bash
sudo mkdir -p /srv/http/autoinstall
```

---

### 1️⃣4️⃣ meta-data

```bash
sudo nano /srv/http/autoinstall/meta-data
```

```yaml
instance-id: node-01
local-hostname: node-01
```

---

### 1️⃣5️⃣ user-data

```bash
sudo nano /srv/http/autoinstall/user-data
```

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: node-01
    username: ubuntu
    password: "$6$HASH"
  ssh:
    install-server: true
    authorized-keys:
      - ssh-ed25519 AAAA...
  storage:
    layout:
      name: lvm
```

Test:

```bash
curl http://192.168.115.129/autoinstall/user-data
```

---

## VIII️⃣ PHASE 7 — DHCP RESERVATION (KHÔNG ĐỔI IP)

```bash
sudo nano /etc/dnsmasq.d/reservation.conf
```

```conf
dhcp-host=AA:BB:CC:DD:EE:01,node-01,192.168.115.11
dhcp-host=AA:BB:CC:DD:EE:02,node-02,192.168.115.12
```

Restart:

```bash
sudo systemctl restart dnsmasq
```

---

## IX️⃣ FLOW THỰC TẾ (UEFI)

1. Máy bật **UEFI PXE**
2. DHCP → IP + `grubx64.efi`
3. TFTP → GRUB EFI
4. GRUB đọc `grub.cfg`
5. Load kernel + initrd
6. Fetch ISO qua HTTP
7. cloud-init autoinstall
8. Reboot → SSH

---

## X️⃣ DEBUG NHANH (UEFI)

```bash
journalctl -u dnsmasq -f
```

```bash
tcpdump -i ens33 port 67 or port 69
```

```bash
tftp 192.168.115.129 -c get grubx64.efi
```

---

## 🔒 CHỐT LẠI

* **UEFI PXE = GRUB EFI**
* **KHÔNG pxelinux**
* **KHÔNG BIOS**
* Flow sạch, tách lớp, debug từng tầng

---