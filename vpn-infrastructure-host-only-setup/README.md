# 🔌 VPN Infrastructure — Host-Only Adapter Setup sebagai Pintu Masuk VPN

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/RGt3BEpqCAU?si=qdeSMlF-svMiAlaM)  
> **Kategori:** `VPN, Network Infrastructure, VirtualBox`

---

## 📋 Deskripsi

Lab ini mengkonfigurasi **Host-Only Adapter** sebagai jalur komunikasi langsung antara komputer fisik dan OPNsense di VirtualBox. Ini adalah langkah fondasi sebelum membangun tunnel VPN — tanpa jalur ini, komputer fisik tidak bisa menjangkau OPNsense sama sekali.

---

## 🎬 Video Tutorial

[![VPN Infrastructure - Host-Only Adapter Setup](https://img.youtube.com/vi/RGt3BEpqCAU/maxresdefault.jpg)](https://youtu.be/RGt3BEpqCAU?si=qdeSMlF-svMiAlaM)

> 📺 **[Tonton di YouTube → VPN Infrastructure: Host-Only Adapter Setup](https://youtu.be/RGt3BEpqCAU?si=qdeSMlF-svMiAlaM)**

---

## 🗺️ Topologi Lab

```
┌─────────────────────────────────────────────────────────────┐
│                    Komputer Fisik Windows                   │
│                    (WiFi — Internet)                        │
│                           │                                 │
│                    VirtualBox Host                          │
│                           │                                 │
│              ┌────────────────────────┐                     │
│              │       OPNsense         │                     │
│              │                        │                     │
│              │  Adapter 1 : NAT       │ ← Internet          │
│              │  Adapter 2 : Internal  │ ← 10.200.200.254    │
│              │  Adapter 3 : Host-Only │ ← 192.168.56.2 ★   │
│              └────────────────────────┘                     │
│                           │                                 │
│            Internal Network (10.200.200.0/24)               │
│      ┌────────┬───────────┼───────────┐                     │
│      ▼        ▼           ▼           ▼                     │
│  Windows 10  Win Server  Kali Linux  SIEM Ubuntu (Wazuh)    │
└─────────────────────────────────────────────────────────────┘

★ Adapter 3 (Host-Only) = jalur baru yang kita tambahkan
  Komputer Fisik (192.168.56.1) ↔ OPNsense (192.168.56.2)
```

---

## ❓ Mengapa Host-Only Adapter Diperlukan?

### Masalah

```
Komputer Fisik
      │
      │ NAT (Adapter 1) — satu arah!
      ▼
  OPNsense ← VM bisa keluar ke internet
             tapi komputer fisik TIDAK BISA masuk ke VM
```

**Adapter 1 (NAT)** bersifat satu arah — VM bisa keluar ke internet melalui komputer fisik, tetapi komputer fisik tidak bisa masuk ke VM. Ibarat gedung yang tidak memiliki alamat publik — dari luar tidak terlihat.

---

### Solusi — Host-Only Adapter

```
Komputer Fisik (192.168.56.1)
      │
      │ Host-Only (Adapter 3) — dua arah! ✅
      ▼
  OPNsense (192.168.56.2)
```

Adapter ketiga ditambahkan khusus sebagai **jalur privat** antara komputer fisik dan OPNsense. Bukan jalur internet, melainkan koneksi langsung antara keduanya. Setelah jalur ini tersedia, tunnel VPN dapat dibangun melewatinya.

---

### Perbedaan dengan Industri

> 💡 Di industri, trik ini tidak diperlukan karena firewall atau VPN gateway memiliki **IP Public** yang dapat dijangkau langsung dari mana saja di internet. Host-Only Adapter adalah **solusi khusus untuk keterbatasan lab VirtualBox**.

---

## 📊 Alokasi IP

| Perangkat | Interface | IP Address |
|-----------|-----------|------------|
| **Komputer Fisik** | Host-Only | `192.168.56.1` (otomatis oleh VirtualBox) |
| **OPNsense** | Adapter 3 (HOSTONLY) | `192.168.56.2` |
| **OPNsense** | Adapter 2 (LAN) | `10.200.200.254` |

---

## ⚙️ Langkah-Langkah Konfigurasi

### Langkah 1 — Cek VirtualBox Host-Only Network

Pastikan VirtualBox sudah memiliki Host-Only network:

```
VirtualBox → File → Tools → Network Manager
→ Tab: Host-only Networks
```

Pastikan sudah ada entry:

| Field | Nilai |
|-------|-------|
| **Nama** | VirtualBox Host-Only Ethernet Adapter |
| **IP** | `192.168.56.1` |

> 💡 Jika belum ada, klik **Create**. IP `192.168.56.1` di-assign otomatis oleh VirtualBox ke komputer fisik saat Host-Only network dibuat.

---

### Langkah 2 — Tambah Adapter 3 di VM OPNsense

> ⚠️ **Matikan VM OPNsense** terlebih dahulu sebelum mengubah network settings.

```
Klik kanan VM OPNsense → Settings → Network → Adapter 3
```

| Field | Nilai |
|-------|-------|
| **Enable Network Adapter** | ✅ Centang |
| **Attached to** | `Host-only Adapter` |
| **Name** | `VirtualBox Host-Only Ethernet Adapter` |
| **Cable Connected** | ✅ Centang |

Klik **OK** → nyalakan kembali OPNsense.

---

### Langkah 3 — Assign Interface di OPNsense

Login ke dashboard OPNsense:

```
https://10.200.200.254
```

Navigasi ke:

```
Interfaces → Assignments
→ Assign a new interface
```

| Field | Nilai |
|-------|-------|
| **Device** | `em2` (Adapter 3 yang baru ditambahkan) |
| **Description** | `HOSTONLY` |

Klik **Add** → **Save**

> 💡 OPNsense mendeteksi adapter secara urut:
> - `em0` = Adapter 1 (WAN/NAT)
> - `em1` = Adapter 2 (LAN/Internal)
> - `em2` = Adapter 3 (Host-Only) ← yang baru ditambahkan

---

### Langkah 4 — Konfigurasi IP Interface HOSTONLY

```
Interfaces → [HOSTONLY]
```

| Field | Nilai |
|-------|-------|
| **Enable** | ✅ Centang |
| **Description** | `HOSTONLY` |
| **IPv4 Configuration Type** | `Static IPv4` |
| **IPv4 Address** | `192.168.56.2` |
| **Subnet Mask** | `/24` |

Klik **Save** → **Apply changes**

> 💡 IP `192.168.56.2` digunakan karena `192.168.56.1` sudah dipakai komputer fisik secara otomatis oleh VirtualBox.

---

### Langkah 5 — Tambah Firewall Rule di HOSTONLY

Tanpa rule ini, OPNsense akan **memblokir semua traffic** dari komputer fisik ke interface HOSTONLY.

```
Firewall → Rules → HOSTONLY → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Action** | `Pass` |
| **Quick** | ✅ Centang |
| **Interface** | `HOSTONLY` |
| **Direction** | `in` |
| **Protocol** | `any` |
| **Source** | `any` |
| **Destination** | `any` |
| **Description** | `Allow all from HOSTONLY` |

Klik **Save** → **Apply changes**

---

### Langkah 6 — Verifikasi di Interfaces Overview

```
Interfaces → Overview
```

Pastikan interface HOSTONLY menampilkan:

| Field | Nilai yang Diharapkan |
|-------|----------------------|
| **Status** | 🟢 Hijau (up) |
| **Device** | `em2` |
| **IPv4** | `192.168.56.2/24` |

---

### Langkah 7 — Testing dari Komputer Fisik

Buka **Command Prompt** di komputer fisik Windows:

```cmd
ping 192.168.56.2
```

**Output yang diharapkan:**

```
Reply from 192.168.56.2: bytes=32 time<1ms TTL=64
Reply from 192.168.56.2: bytes=32 time<1ms TTL=64
Reply from 192.168.56.2: bytes=32 time<1ms TTL=64
Reply from 192.168.56.2: bytes=32 time<1ms TTL=64
```

> ✅ Jika ping berhasil, jalur komunikasi antara komputer fisik dan OPNsense sudah terbuka — siap untuk membangun tunnel VPN.

---

## 🛠️ Troubleshooting

| Masalah | Kemungkinan Penyebab | Solusi |
|---------|----------------------|--------|
| **Ping timeout** | Adapter 3 belum pakai Host-Only yang benar | Cek Settings → Network → Adapter 3 di VirtualBox |
| **Interface HOSTONLY merah** | Cable Connected tidak dicentang | Matikan OPNsense, centang Cable Connected, nyalakan lagi |
| **IP bentrok** | OPNsense diset ke `192.168.56.1` | Ganti IP OPNsense HOSTONLY ke `192.168.56.2` |

---

## 📌 Kesimpulan

Setelah mengikuti lab ini, kamu telah berhasil:

- ✅ Membuat Host-Only network di VirtualBox
- ✅ Menambahkan Adapter 3 (Host-Only) ke VM OPNsense
- ✅ Mengassign interface HOSTONLY di OPNsense
- ✅ Mengkonfigurasi IP static `192.168.56.2` di interface HOSTONLY
- ✅ Menambahkan firewall rule untuk mengizinkan traffic dari komputer fisik
- ✅ Memverifikasi koneksi dengan ping dari komputer fisik ke OPNsense

Jalur komunikasi antara komputer fisik dan OPNsense sudah terbuka — lanjut ke **Part 3** untuk membangun **Public Key Infrastructure (PKI)** sebagai fondasi autentikasi VPN.

---

## 📚 Referensi

- [OPNsense Interface Documentation](https://docs.opnsense.org/manual/interfaces.html)
- [VirtualBox Host-Only Networking](https://www.virtualbox.org/manual/ch06.html#network_hostonly)
- [OPNsense Firewall Rules](https://docs.opnsense.org/manual/firewall.html)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/RGt3BEpqCAU?si=qdeSMlF-svMiAlaM) | ⭐ Star repo ini jika membantu!*
