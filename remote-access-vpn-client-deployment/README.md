# 🚀 Remote Access VPN — Export, Connect & Testing

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/OPsvTvSN50M?si=GGW6kAfbgBL-VxNA)  
> **Kategori:** `VPN, OpenVPN, Remote Access`

---

## 📋 Deskripsi

Lab ini adalah tahap akhir dari seri VPN — semua yang sudah dibangun di Part 2, 3, dan 4 akhirnya diuji. File `.ovpn` diekspor dari OPNsense, diimport ke **OpenVPN Connect** di Windows, lalu diverifikasi bahwa seluruh VM di lab bisa diakses dari komputer fisik melalui tunnel VPN.

---

## 🎬 Video Tutorial

[![Remote Access VPN - Export, Connect & Testing](https://img.youtube.com/vi/OPsvTvSN50M/maxresdefault.jpg)](https://youtu.be/OPsvTvSN50M?si=GGW6kAfbgBL-VxNA)

> 📺 **[Tonton di YouTube → Remote Access VPN: Export, Connect & Testing](https://youtu.be/OPsvTvSN50M?si=GGW6kAfbgBL-VxNA)**

---

## 🔄 Tiga Tahap di Lab Ini

| Tahap | Penjelasan |
|-------|------------|
| **Export** | OPNsense menghasilkan file `.ovpn` berisi semua informasi koneksi — alamat server, port, sertifikat CA, client certificate, private key, dan TLS key dalam satu file |
| **Connect** | File `.ovpn` diimport ke OpenVPN Connect di Windows — tunnel terenkripsi terbentuk dan komputer fisik mendapat IP dari subnet VPN `10.8.0.x` |
| **Testing** | Verifikasi semua VM di lab bisa diakses, Wazuh dashboard bisa dibuka, dan internet tetap berjalan normal |

---

## 🗺️ Hasil Akhir yang Dicapai

```
Komputer Fisik Windows
IP VPN: 10.8.0.2
        │
        │  Tunnel VPN (10.8.0.0/24)
        ▼
OPNsense (10.8.0.1 / 10.200.200.254)
        │
        │  Route: 10.200.200.0/24
        ▼
Internal Network Lab
├── Windows Server  (10.200.200.20)
├── Windows 10      (10.200.200.30)
├── Kali Linux      (10.200.200.10)
└── Ubuntu SIEM     (10.200.200.100) ← Wazuh Dashboard
```

---

## ⚙️ Langkah-Langkah

### Langkah 1 — Export File .ovpn

```
VPN → OpenVPN → Client Export
```

Isi field berikut:

| Field | Nilai |
|-------|-------|
| **Remote Access Server** | `Lab-VPN-Server udp4:1194` |
| **Export type** | `File Only` |
| **Hostname** | `192.168.56.2` |
| **Port** | `1194` |

Scroll ke bawah ke bagian **Accounts / certificates** — pastikan terlihat:

| Certificate | Linked user |
|-------------|-------------|
| *(none)* | Exclude certificate from export |
| OpenVPN-Server-Cert | — |
| **vpnuser-cert** | **vpnuser** |

Klik ikon **download (ikon awan)** di baris `vpnuser-cert`.

> 💡 File `.ovpn` berisi semua yang dibutuhkan client — sertifikat, private key, TLS key, dan konfigurasi server — semuanya dalam satu file. Simpan di lokasi yang mudah ditemukan, misalnya **Desktop**.

---

### Langkah 2 — Import ke OpenVPN Connect

```
1. Buka OpenVPN Connect di komputer fisik Windows
2. Klik Import Profile → pilih File
3. Browse ke file .ovpn yang baru didownload
4. Klik Add / Import
```

Masukkan credentials:

| Field | Nilai |
|-------|-------|
| **Username** | `vpnuser` |
| **Password** | *(password yang dibuat di Part 3)* |

Klik **Connect**.

---

### Langkah 3 — Verifikasi Tunnel Terbentuk

Buka **Command Prompt** di komputer fisik:

```cmd
ipconfig
```

Cari adapter baru bernama **TAP-Windows** atau **OpenVPN**:

```
TAP-Windows Adapter:
   IPv4 Address : 10.8.0.2
   Subnet Mask  : 255.255.255.0
```

> ✅ Munculnya IP `10.8.0.x` adalah tanda bahwa tunnel VPN sudah terbentuk dan komputer fisik sudah "masuk" ke jaringan lab.

---

### Langkah 4 — Cek Route Table

Masih di CMD, jalankan:

```cmd
route print
```

Pastikan ada route berikut:

```
10.200.200.0   255.255.255.0   10.8.0.1   10.8.0.2
```

**Artinya:** Untuk menjangkau jaringan lab `10.200.200.0/24`, traffic akan lewat OPNsense di `10.8.0.1` melalui tunnel VPN. Inilah yang memungkinkan komputer fisik bisa akses semua VM di lab.

> ⚠️ **Pastikan TIDAK ada** route `0.0.0.0` yang mengarah ke tunnel — itu tanda `redirect-gateway` aktif dan akan membuat internet tidak bisa dipakai.

---

### Langkah 5 — Testing Koneksi ke Semua VM

Buka CMD dan ping ke semua VM satu per satu:

```cmd
:: OPNsense LAN — harus reply pertama kali
ping 10.200.200.254

:: SIEM Ubuntu (Wazuh)
ping 10.200.200.100

:: Windows Server
ping 10.200.200.20

:: Windows 10
ping 10.200.200.30

:: Kali Linux
ping 10.200.200.10
```

**Output yang diharapkan:**

```
Reply from 10.200.200.x: bytes=32 time<10ms TTL=64
Reply from 10.200.200.x: bytes=32 time<10ms TTL=64
Reply from 10.200.200.x: bytes=32 time<10ms TTL=64
```

---

### Langkah 6 — Akses Wazuh Dashboard via VPN

Buka browser di komputer fisik dan akses:

```
https://10.200.200.100
```

Akan muncul warning **"Your connection is not private"** — ini **normal** karena Wazuh menggunakan sertifikat self-signed.

```
Klik Advanced → Proceed to 10.200.200.100
```

Login dengan:

| Field | Nilai |
|-------|-------|
| **Username** | `admin` |
| **Password** | *(password Wazuh kamu)* |

> ✅ Dashboard Wazuh berhasil terbuka dari komputer fisik melalui tunnel VPN — ini membuktikan seluruh pipeline VPN dari Part 2 hingga Part 5 berjalan dengan benar.

---

## 📊 Checklist Verifikasi Akhir

| Verifikasi | Perintah / Cara | Hasil yang Diharapkan |
|------------|-----------------|----------------------|
| Tunnel terbentuk | `ipconfig` | IP `10.8.0.x` di adapter TAP-Windows |
| Route lab tersedia | `route print` | Route `10.200.200.0` via `10.8.0.1` |
| OPNsense reachable | `ping 10.200.200.254` | Reply |
| Wazuh reachable | `ping 10.200.200.100` | Reply |
| Windows Server reachable | `ping 10.200.200.20` | Reply |
| Windows 10 reachable | `ping 10.200.200.30` | Reply |
| Kali Linux reachable | `ping 10.200.200.10` | Reply |
| Wazuh Dashboard | Browser → `https://10.200.200.100` | Dashboard terbuka |
| Internet tetap jalan | `ping 8.8.8.8` | Reply |

---

## 🛠️ Troubleshooting

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| **OpenVPN Connect error saat import** | File `.ovpn` corrupt atau tidak lengkap | Re-export dari OPNsense, pastikan vpnuser-cert yang didownload |
| **Tidak dapat IP 10.8.0.x** | Server belum aktif atau port 1194 diblokir | Cek OPNsense → VPN → OpenVPN → Log File |
| **Ping ke VM gagal** | Route tidak terbentuk | Cek `route print`, pastikan ada route `10.200.200.0` |
| **Internet mati setelah connect** | `redirect-gateway` aktif di config | Cek file `.ovpn`, hapus baris `redirect-gateway def1` jika ada |
| **Wazuh tidak bisa dibuka** | Firewall rule OpenVPN belum ada | Cek Firewall → Rules → OpenVPN, tambah rule Pass |
| **Authentication failed** | Username/password salah | Pastikan username `vpnuser` dan password sesuai Part 3 |

---

## 📌 Kesimpulan

Setelah mengikuti lab ini, kamu telah berhasil:

- ✅ Mengekspor file `.ovpn` dari OPNsense Client Export
- ✅ Mengimport dan mengkoneksikan OpenVPN Connect di Windows
- ✅ Memverifikasi tunnel VPN terbentuk dengan IP `10.8.0.2`
- ✅ Memverifikasi route ke jaringan lab tersedia
- ✅ Mengakses semua VM di lab dari komputer fisik via VPN
- ✅ Membuka Wazuh Dashboard dari komputer fisik via VPN

**Seluruh seri VPN Lab telah selesai** — dari Host-Only setup, PKI, OpenVPN server, hingga client deployment dan testing. 🎉

---

## 📚 Referensi

- [OpenVPN Connect Documentation](https://openvpn.net/client/)
- [OPNsense Client Export Documentation](https://docs.opnsense.org/manual/vpn.html#client-export)
- [Wazuh Documentation](https://documentation.wazuh.com/)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/OPsvTvSN50M?si=GGW6kAfbgBL-VxNA) | ⭐ Star repo ini jika membantu!*
