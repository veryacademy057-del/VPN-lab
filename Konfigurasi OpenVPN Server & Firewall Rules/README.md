# ⚙️ Konfigurasi OpenVPN Server & Firewall Rules

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/DxU-Dv5KI0I?si=8hMCTN_GZw6llekS)  
> **Kategori:** `VPN, OpenVPN, Firewall Configuration`

---

## 📋 Deskripsi

Lab ini merakit semua komponen PKI yang sudah dibuat di Part 3 menjadi **VPN server yang aktif berjalan**. Mencakup konfigurasi OpenVPN Instance, instalasi plugin Client Export, dan tiga firewall rules yang mengontrol seluruh traffic VPN di OPNsense.

---

## 🎬 Video Tutorial

[![Konfigurasi OpenVPN Server & Firewall Rules](https://img.youtube.com/vi/DxU-Dv5KI0I/maxresdefault.jpg)](https://youtu.be/DxU-Dv5KI0I?si=8hMCTN_GZw6llekS)

> 📺 **[Tonton di YouTube → Konfigurasi OpenVPN Server & Firewall Rules](https://youtu.be/DxU-Dv5KI0I?si=8hMCTN_GZw6llekS)**

---

## 🧱 Dua Bagian Utama

| Bagian | Keterangan |
|--------|------------|
| **OpenVPN Instance** | Konfigurasi server VPN — protocol, port, sertifikat, subnet tunnel, dan jaringan yang bisa diakses client |
| **Firewall Rules** | Tiga aturan yang mengontrol traffic VPN di tiga interface berbeda |

---

## ❓ Kenapa Butuh Tiga Firewall Rule?

OPNsense secara default menerapkan prinsip **default deny** — semua traffic yang tidak diizinkan secara eksplisit akan diblokir. Setiap interface memiliki policy sendiri sehingga setiap jalur traffic perlu rule tersendiri.

| Interface | Fungsi Rule |
|-----------|-------------|
| **WAN** | Membuka port 1194 UDP agar koneksi VPN bisa masuk dari jaringan luar |
| **HOSTONLY** | Mengizinkan komputer fisik konek ke port VPN melalui jalur Host-Only. Tanpa rule ini, meskipun ping sudah bisa, koneksi VPN tetap diblok |
| **OpenVPN** | Mengizinkan client yang sudah terkoneksi VPN untuk mengakses VM-VM di lab. Tanpa rule ini, tunnel terbentuk tapi tidak bisa akses jaringan internal |

---

## 🗺️ Arsitektur Traffic VPN

```
Komputer Fisik (192.168.56.1)
        │
        │  UDP 1194 (via Host-Only)
        ▼
┌───────────────────────────────────────┐
│            OPNsense                   │
│                                       │
│  HOSTONLY rule → izinkan port 1194    │
│  WAN rule      → izinkan port 1194    │
│                                       │
│  OpenVPN Instance                     │
│  └── Bind: 192.168.56.2:1194         │
│  └── Tunnel: 10.8.0.0/24            │
│  └── Route: 10.200.200.0/24         │
│                                       │
│  OpenVPN rule → izinkan akses lab     │
└───────────────────────────────────────┘
        │
        │  10.8.0.0/24 (VPN tunnel)
        ▼
Internal Network (10.200.200.0/24)
├── Windows Server  (.20)
├── Windows 10      (.30)
├── Kali Linux      (.10)
└── Ubuntu SIEM     (.100)
```

---

## ⚙️ Langkah-Langkah Konfigurasi

### Langkah 1 — Konfigurasi OpenVPN Instance

```
VPN → OpenVPN → Instances → klik + (Add)
```

**General Settings:**

| Field | Nilai |
|-------|-------|
| **Role** | `Server` |
| **Description** | `Lab-VPN-Server` |
| **Enabled** | ✅ Centang |
| **Protocol** | `UDP4` ⚠️ bukan UDP biasa! |
| **Port number** | `1194` |
| **Bind address** | `192.168.56.2` ← IP HOSTONLY OPNsense |
| **Type** | `tun` |
| **Server (IPv4)** | `10.8.0.0/24` |
| **Keep alive interval** | `10` |
| **Keep alive timeout** | `120` |

> ⚠️ **Penting:** Pastikan Protocol diisi **`UDP4`** bukan `UDP`. Jika diisi `UDP`, OpenVPN akan mencoba bind ke IPv6 dan muncul error `Could not determine IPv4/IPv6 protocol` di log.

**Trust:**

| Field | Nilai |
|-------|-------|
| **Certificate** | `OpenVPN-Server-Cert` |
| **Certificate Authority** | `Lab VPN-CA` |
| **Verify Client Certificate** | `Require valid certificate` |
| **TLS static key** | Klik **Generate** |

**Authentication:**

| Field | Nilai |
|-------|-------|
| **Authentication** | `Local Database` |

**Routing:**

| Field | Nilai |
|-------|-------|
| **Local Network** | `10.200.200.0/24` |

Klik **Save** → **Apply**.

**Verifikasi:**
```
VPN → OpenVPN → Instances
→ Lab-VPN-Server muncul di daftar
→ Role: Server
→ Type: TUN
→ Enabled: ✅
```

---

### Langkah 2 — Install Plugin Client Export

Plugin ini dibutuhkan untuk mengekspor file `.ovpn` yang akan digunakan di Part 5.

```
System → Firmware → Plugins
→ Ketik "openvpn" di kolom search
→ Centang "Show community plugins"
→ Cari: os-openvpn-legacy
→ Klik + untuk install
```

Tunggu hingga selesai lalu **refresh browser**.

**Verifikasi:**
```
VPN → OpenVPN
→ Menu "Client Export" muncul ✅
```

---

### Langkah 3 — Firewall Rule di WAN

Rule ini membuka port VPN dari jaringan luar ke OPNsense.

```
Firewall → Rules → WAN → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Action** | `Pass` |
| **Quick** | ✅ Centang |
| **Interface** | `WAN` |
| **Direction** | `in` |
| **TCP/IP Version** | `IPv4` |
| **Protocol** | `UDP` |
| **Source** | `any` |
| **Destination** | `WAN address` |
| **Destination port from** | `1194` |
| **Destination port to** | `1194` |
| **Description** | `Allow OpenVPN UDP 1194` |

Klik **Save** → **Apply changes**.

---

### Langkah 4 — Firewall Rule di HOSTONLY

Rule ini mengizinkan komputer fisik konek ke VPN server melalui interface Host-Only.

```
Firewall → Rules → HOSTONLY → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Action** | `Pass` |
| **Quick** | ✅ Centang |
| **Interface** | `HOSTONLY` |
| **Direction** | `in` |
| **TCP/IP Version** | `IPv4` |
| **Protocol** | `UDP` |
| **Source** | `any` |
| **Destination** | `HOSTONLY address` |
| **Destination port from** | `1194` |
| **Destination port to** | `1194` |
| **Description** | `Allow OpenVPN from HOSTONLY` |

Klik **Save** → **Apply changes**.

---

### Langkah 5 — Firewall Rule di OpenVPN

Rule ini mengizinkan client VPN yang sudah terkoneksi untuk mengakses semua VM di jaringan internal lab.

```
Firewall → Rules → OpenVPN → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Action** | `Pass` |
| **Quick** | ✅ Centang |
| **Interface** | `OpenVPN` |
| **Direction** | `in` |
| **TCP/IP Version** | `IPv4` |
| **Protocol** | `any` |
| **Source** | `any` |
| **Destination** | `any` |
| **Description** | `Allow VPN clients to lab` |

Klik **Save** → **Apply changes**.

---

## ✅ Verifikasi Akhir

### Cek OpenVPN Server Berjalan

```
VPN → OpenVPN → Log File
```

Pastikan tidak ada error fatal. Warning seperti `script-security` dan `keepalive` adalah **normal dan bisa diabaikan**.

---

### Cek Interface OpenVPN Aktif

```
Interfaces → Overview
→ Pastikan ada interface ovpns1
→ IP: 10.8.0.1 ✅
```

> ✅ Munculnya `ovpns1` dengan IP `10.8.0.1` adalah tanda bahwa OpenVPN server sudah aktif dan siap menerima koneksi.

---

## 📊 Ringkasan Firewall Rules

| Interface | Protocol | Port | Tujuan |
|-----------|----------|------|--------|
| **WAN** | UDP | 1194 | Membuka port VPN dari internet |
| **HOSTONLY** | UDP | 1194 | Mengizinkan komputer fisik konek VPN lewat Host-Only adapter |
| **OpenVPN** | any | any | Mengizinkan client VPN mengakses seluruh jaringan lab internal |

---

## 🛠️ Troubleshooting

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| **Server tidak muncul di Instances** | Belum klik Apply setelah Save | Klik tombol Apply di halaman Instances |
| **Log error `AF_INET6`** | Protocol diisi `UDP` bukan `UDP4` | Edit Instance, ganti Protocol ke `UDP4` |
| **Log error `could not determine`** | Bind address kosong | Isi Bind address dengan `192.168.56.2` |
| **`ovpns1` tidak muncul di Overview** | Server belum aktif karena error certificate | Cek log, pastikan CA dan certificate sudah benar |
| **Client konek tapi tidak bisa akses VM** | Rule di OpenVPN interface belum ada | Tambah rule Pass di Firewall → Rules → OpenVPN |
| **Connection timeout dari client** | Bind address salah atau rule HOSTONLY belum ada | Pastikan bind ke `192.168.56.2` dan rule HOSTONLY sudah di-apply |

---

## 📌 Kesimpulan

Setelah mengikuti lab ini, kamu telah berhasil:

- ✅ Mengkonfigurasi OpenVPN Instance dengan PKI dari Part 3
- ✅ Menginstall plugin Client Export (`os-openvpn-legacy`)
- ✅ Menambahkan firewall rule di interface WAN
- ✅ Menambahkan firewall rule di interface HOSTONLY
- ✅ Menambahkan firewall rule di interface OpenVPN
- ✅ Memverifikasi OpenVPN server aktif dengan interface `ovpns1`

OpenVPN server sudah aktif dan semua firewall rules ter-apply — lanjut ke **Part 5** untuk export file `.ovpn`, import ke OpenVPN Connect di Windows, dan testing koneksi ke seluruh lab.

---

## 📚 Referensi

- [OPNsense OpenVPN Documentation](https://docs.opnsense.org/manual/vpn.html)
- [OpenVPN Protocol Reference](https://openvpn.net/community-resources/reference-manual-for-openvpn-2-4/)
- [OPNsense Firewall Rules](https://docs.opnsense.org/manual/firewall.html)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/DxU-Dv5KI0I?si=8hMCTN_GZw6llekS) | ⭐ Star repo ini jika membantu!*
