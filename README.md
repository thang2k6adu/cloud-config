## 0️⃣ Tiền đề (check trước khi làm)

Trên PXE server:

```bash
ip a
```

Xác nhận:

* NIC đúng: `ens33`
* IP đúng: `192.168.115.129/24`

Nếu **NIC khác tên → sửa toàn bộ config theo NIC đó**.

---

## I️⃣ PHASE 1 — CÀI & CẤU HÌNH DHCP + PXE

### 1️⃣ Cài dnsmasq

```bash
sudo apt update
sudo apt install -y dnsmasq
```

Kiểm tra dnsmasq chạy:

```bash
systemctl status dnsmasq --no-pager
```

---

### 2️⃣ Tạo file cấu hình PXE BIOS

```bash
sudo nano /etc/dnsmasq.d/pxe-bios.conf
```

**DÁN NGUYÊN KHỐI:**

```conf
interface=ens33
bind-interfaces

dhcp-range=192.168.115.50,192.168.115.100,12h
dhcp-authoritative

dhcp-option=3,192.168.115.1
dhcp-option=6,192.168.115.129

; Nó nói với client:
; “Mỗi khi mày hỏi tên KHÔNG CÓ DẤU CHẤM
; → tự động thử thêm .lab.local”
; ping node-01 -> thêm .lab.local -> node-01.lab.local -> nếu ko có thì ko tự thêm và lỗi
dhcp-option=15,lab.local
domain=lab.local
expand-hosts

enable-tftp
tftp-root=/srv/tftp

dhcp-boot=pxelinux.0
pxe-service=x86PC,"Install Ubuntu (BIOS PXE)",pxelinux.0
```

Lưu → thoát.

---

### 3️⃣ Test & restart dnsmasq

```bash
sudo dnsmasq --test
```

Nếu thấy:

```
dnsmasq: syntax check OK
```

→ restart:

```bash
sudo systemctl restart dnsmasq
```

Log realtime:

```bash
journalctl -u dnsmasq -f
```

---

## II️⃣ PHASE 2 — TFTP BOOTLOADER (SYSLINUX)

### 4️⃣ Cài syslinux

```bash
sudo apt install -y syslinux-common
```

---

### 5️⃣ Tạo cấu trúc TFTP

```bash
sudo mkdir -p /srv/tftp/pxelinux.cfg
```

---

### 6️⃣ Copy bootloader BIOS

```bash
sudo cp /usr/lib/PXELINUX/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/modules/bios/ldlinux.c32 /srv/tftp/
```

Kiểm tra:

```bash
ls -lh /srv/tftp
```

Phải thấy:

```
pxelinux.0
ldlinux.c32
pxelinux.cfg/
```

---

### 7️⃣ Test TFTP thủ công

```bash
tftp 192.168.115.129
tftp> get pxelinux.0
tftp> quit
```

Nếu **get được file → TFTP OK**.

---

## III️⃣ PHASE 3 — KERNEL + INITRD

### 8️⃣ Chuẩn bị HTTP thư mục ISO

```bash
sudo mkdir -p /srv/http/ubuntu
```

Copy ISO vào (từ scp / USB / WinSCP):

```bash
sudo cp ubuntu-22.04.5-live-server-amd64.iso /srv/http/ubuntu/
```

---

### 9️⃣ Mount ISO

```bash
sudo mkdir -p /mnt/iso
sudo mount -o loop /srv/http/ubuntu/ubuntu-22.04.5-live-server-amd64.iso /mnt/iso
```

Check:

```bash
ls /mnt/iso/casper
```

Phải thấy:

```
vmlinuz
initrd
```

---

### 🔟 Copy kernel + initrd

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

## IV️⃣ PHASE 4 — PXELINUX MENU (CỰC QUAN TRỌNG)

### 1️⃣1️⃣ Tạo menu mặc định

```bash
sudo nano /srv/tftp/pxelinux.cfg/default
```

**DÁN NGUYÊN KHỐI:**

```cfg
DEFAULT install
PROMPT 0
TIMEOUT 50

LABEL install
  KERNEL ubuntu/vmlinuz
  INITRD ubuntu/initrd
  APPEND ip=dhcp \
         url=http://192.168.115.129/ubuntu/ubuntu-22.04.5-live-server-amd64.iso \
         autoinstall \
         ds=nocloud-net;s=http://192.168.115.129/autoinstall/ ---
```

👉 **Dòng APPEND = não của hệ thống**

---

## V️⃣ PHASE 5 — HTTP SERVER

### 1️⃣2️⃣ Cài nginx

```bash
sudo apt install -y nginx
```

---

### 1️⃣3️⃣ Xoá site mặc định

```bash
sudo rm /etc/nginx/sites-enabled/default
```

---

### 1️⃣4️⃣ Tạo site PXE

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

Enable site:

```bash
sudo ln -s /etc/nginx/sites-available/pxe /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

Test:

```bash
curl http://192.168.115.129/ubuntu/
```

---

## VI️⃣ PHASE 6 — AUTOINSTALL (cloud-init)

### 1️⃣5️⃣ Tạo thư mục

```bash
sudo mkdir -p /srv/http/autoinstall
```

---

### 1️⃣6️⃣ meta-data

```bash
sudo nano /srv/http/autoinstall/meta-data
```

```yaml
instance-id: node-01
local-hostname: node-01
```

---

### 1️⃣7️⃣ user-data

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

## VII️⃣ PHASE 7 — DHCP RESERVATION

### 1️⃣8️⃣ Tạo file reservation

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

## VIII️⃣ FLOW THỰC TẾ (BIOS)

1. Máy bật **Legacy PXE**
2. DHCP cấp IP + `pxelinux.0`
3. TFTP gửi bootloader
4. pxelinux đọc `pxelinux.cfg/default`
5. Kernel + initrd load
6. ISO fetch qua HTTP
7. cloud-init chạy autoinstall
8. Reboot → SSH

---

## IX️⃣ DEBUG NHANH (SENIOR HAY DÙNG)

```bash
journalctl -u dnsmasq -f
```

```bash
tcpdump -i ens33 port 67 or port 69
```

```bash
ls -lh /srv/tftp
```

---

## 🔒 CHỐT CUỐI

* BIOS PXE = **SYSLINUX**
* **GRUB = KHÔNG TỒN TẠI**
* **UEFI = CHƯA ĐƯỢC PHÉP**

---



interface=ens33
bind-interfaces

dhcp-range=192.168.115.50,192.168.115.100,12h
dhcp-authoritative

dhcp-option=3,192.168.115.1
dhcp-option=6,192.168.115.129
dhcp-option=15,lab.local

enable-tftp
tftp-root=/srv/tftp

# Nhận diện loại firmware
dhcp-match=set:bios,option:client-arch,0
dhcp-match=set:uefi,option:client-arch,7

# BIOS → pxelinux
dhcp-boot=tag:bios,pxelinux.0

# UEFI → GRUB
dhcp-boot=tag:uefi,grubx64.efi

pxe-service=tag:bios,x86PC,"Install Ubuntu (BIOS PXE)",pxelinux.0
pxe-service=tag:uefi,UEFI,"Install Ubuntu (UEFI PXE)",grubx64.efi
