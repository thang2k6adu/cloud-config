🧱 BỐI CẢNH

Có 10 cục sắt trống (bare metal)

CHƯA CÓ OS

CHƯA CÓ CONTROL PLANE

CHƯA CÓ GÌ HẾT

👉 Focus 1 con đầu tiên là ĐÚNG 100%

Con đầu tiên này KHÔNG PHẢI worker
👉 Nó là SEED / CONTROL / BOOTSTRAP NODE

✅ CHECKLIST CÀI CON ĐẦU TIÊN (CHUẨN SENIOR)
🥇 PHASE 1 — HẠ TẦNG TỐI THIỂU (BẮT BUỘC)
1️⃣ BIOS / Firmware

 UEFI mode (không legacy)

 PXE boot enabled

 NIC thứ tự cố định

 RAID / disk mode xác định

❌ Nếu BIOS loạn → infra sau này nát

2️⃣ Network vật lý

 Cắm vào management network

 Switch port fixed VLAN

 Ghi lại MAC address

🔑 MAC = identity của server bare metal

🥈 PHASE 2 — NETWORK LOGIC (CỰC QUAN TRỌNG)
3️⃣ IP TĨNH

Con đầu tiên BẮT BUỘC IP TĨNH

 IP tĩnh (vd: 10.0.0.10)

 Gateway

 DNS (ban đầu có thể là 8.8.8.8)

👉 KHÔNG DHCP

4️⃣ Hostname chuẩn

 infra-01 / control-01 / bootstrap-01

 KHÔNG dùng node-1 chung chung

🥉 PHASE 3 — OS BASELINE
5️⃣ OS

 Ubuntu Server LTS

 LVM

 Timezone

 Locale

👉 Phần này file của mày OK

6️⃣ User & SSH

 User riêng (không root)

 SSH key only

 Root login disabled

 SSH port xác định

👉 File của mày OK

🟦 PHASE 4 — BẢO MẬT CƠ BẢN
7️⃣ Security baseline

 UFW default deny

 SSH allow list

 Fail2ban

 Swap off (nếu định k8s)

👉 File của mày OK

🟩 PHASE 5 — ĐÂY LÀ CHỖ FILE MÀY THIẾU HOÀN TOÀN
8️⃣ DỊCH VỤ CONTROL (BẮT BUỘC)

Con đầu tiên PHẢI CÀI THÊM:

 DHCP server

 DNS server

 HTTP server

 (tuỳ) TFTP / PXE

Ví dụ:

dnsmasq / isc-dhcp
bind / dnsmasq
nginx


👉 Đây là thứ giúp:

Các node khác tự nhận IP

Mày KHÔNG PHẢI NHỚ IP

SSH bằng hostname

9️⃣ DHCP reservation

 MAC → IP cố định

 MAC → hostname

AA:BB:CC → node-01 → 10.0.0.11
AA:BB:DD → node-02 → 10.0.0.12


👉 VẪN DHCP – NHƯNG KHÔNG BAO GIỜ ĐỔI IP

🔟 DNS nội bộ

 node-01 → 10.0.0.11

 node-02 → 10.0.0.12

👉 Từ đây:

ssh node-01

🟨 PHASE 6 — QUẢN LÝ
1️⃣1️⃣ Inventory

 File YAML / Ansible inventory

 Mapping:

node-01:
  ip: 10.0.0.11
  role: worker

1️⃣2️⃣ SSH config
Host node-*
  User thang2k6adu
  Port 8022


👉 Không cần nhớ user / port


OK, bỏ hết chửi bậy sang một bên. Tao **đi từ tư duy → kiến trúc → từng bước làm**, đúng kiểu **PXE server “chuẩn bài” mà senior dùng**, không nhảy cóc.

---

# I. TƯ DUY CHUẨN VỀ PXE (CỐT LÕI)

PXE **KHÔNG PHẢI** là “1 tool”, mà là **4 dịch vụ ghép lại**:

```
[ Client trống ]
   │
   │ 1. DHCP  → hỏi: boot bằng gì?
   │
   ▼
[ PXE Server ]
   ├─ DHCP   (trả IP + bootloader)
   ├─ TFTP   (đưa bootloader + kernel + initrd)
   ├─ HTTP   (đưa ISO + autoinstall)
   └─ Installer logic (grub/ipxe config)
```

👉 **PXE chỉ lo BOOT**
👉 **Autoinstall lo CÀI**

---

# II. KIẾN TRÚC CHUẨN (NÊN DÙNG)

Giả sử PXE server của mày:

```
IP: 192.168.115.129
OS: Ubuntu Server 22.04
```

Thư mục chuẩn:

```
/srv/tftp/            ← TFTP root
/srv/http/            ← HTTP root
```

---

# III. BƯỚC 1 – SETUP DHCP (QUAN TRỌNG NHẤT)

## Trường hợp A (chuẩn nhất): PXE = DHCP luôn

Dùng **dnsmasq** (đơn giản, senior rất hay dùng).

```bash
sudo apt update
sudo apt install -y dnsmasq
```

### Cấu hình `/etc/dnsmasq.d/pxe.conf`

```conf
# Bind NIC
interface=ens33
bind-interfaces

# DHCP range
dhcp-range=192.168.115.50,192.168.115.100,12h

# Gateway + DNS
dhcp-option=3,192.168.115.1
dhcp-option=6,192.168.115.129

# PXE
enable-tftp
tftp-root=/srv/tftp

# 👇 DÒNG QUAN TRỌNG NHẤT
dhcp-boot=pxelinux.0

pxe-service=x86PC,"Install Ubuntu via PXE",pxelinux.0
```

Restart:

```bash
sudo systemctl restart dnsmasq
```

---

# IV. BƯỚC 2 – TFTP (BOOTLOADER + KERNEL)

## Tạo thư mục

```bash
sudo mkdir -p /srv/tftp/{grub,ubuntu}
```

## Copy GRUB UEFI

```bash
sudo apt install -y grub-efi-amd64-bin
sudo cp /usr/lib/grub/x86_64-efi/grubx64.efi /srv/tftp/
```

---

# V. BƯỚC 3 – KERNEL + INITRD (TỪ ISO)

Mày **đã tải ISO sẵn trên Windows** → chép sang PXE server, ví dụ:

```bash
scp ubuntu-22.04.5-live-server-amd64.iso thang2k6adu@192.168.115.129:/srv/http/ubuntu/
```

Mount ISO:

```bash
sudo mkdir /mnt/iso
sudo mount -o loop /srv/http/ubuntu/ubuntu-22.04.5-live-server-amd64.iso /mnt/iso
```

Copy kernel/initrd:

```bash
sudo cp /mnt/iso/casper/vmlinuz /srv/tftp/ubuntu/
sudo cp /mnt/iso/casper/initrd /srv/tftp/ubuntu/
sudo umount /mnt/iso
```

---

# VI. BƯỚC 4 – HTTP SERVER (ISO + AUTOINSTALL)

```bash
sudo apt install -y nginx
```

### Cấu trúc:

```
/srv/http/
├── ubuntu/
│   └── ubuntu-22.04.5-live-server-amd64.iso
└── autoinstall/
    ├── meta-data
    └── user-data
```

Nginx root:

```bash
sudo rm /etc/nginx/sites-enabled/default
```

`/etc/nginx/sites-available/pxe`:

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

---

# VII. BƯỚC 5 – AUTOINSTALL (cloud-init)

## `/srv/http/autoinstall/meta-data`

```yaml
instance-id: node-1
local-hostname: node-1
```

## `/srv/http/autoinstall/user-data`

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: node-1
    username: ubuntu
    password: "$6$HASH"
  ssh:
    install-server: true
    authorized-keys:
      - ssh-ed25519 AAAA...
```

---

# VIII. BƯỚC 6 – INSTALLER LOGIC (CỐT LÕI)

## `/srv/tftp/grub/grub.cfg`

```cfg
set timeout=5

menuentry "Ubuntu 22.04 Autoinstall" {
    linux /ubuntu/vmlinuz \
        ip=dhcp \
        url=http://192.168.115.129/ubuntu/ubuntu-22.04.5-live-server-amd64.iso \
        autoinstall \
        ds=nocloud-net;s=http://192.168.115.129/autoinstall/
    initrd /ubuntu/initrd
}
```

👉 **Đây chính là chỗ mày hỏi “biết kiểu gì”**
→ **là kernel cmdline**

---

# IX. FLOW HOẠT ĐỘNG (NHỚ CHO KỸ)

1. Máy trống bật PXE
2. DHCP cấp IP + bootloader
3. TFTP gửi GRUB
4. GRUB load kernel + initrd
5. Kernel fetch ISO qua HTTP
6. cloud-init fetch autoinstall
7. OS cài xong → reboot → SSH vào

---

# X. CHECKLIST PXE CHUẨN

* [ ] DHCP trả đúng bootloader
* [ ] TFTP đọc được kernel
* [ ] HTTP truy cập được ISO
* [ ] autoinstall hợp lệ YAML
* [ ] grub.cfg đúng IP

---

Nếu mày muốn **bước tiếp theo đúng level senior**:

* iPXE (script hoá)
* 1 PXE → nhiều role (control / worker)
* Gán hostname theo MAC
* Debug PXE treo

👉 nói **“tiếp level senior”** là tao làm tiếp.
