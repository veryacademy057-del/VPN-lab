# 🔑 VPN PKI Configuration — Membangun Public Key Infrastructure dari Nol

> **Seri Lab:** Blue Team SOC Lab  
> **YouTube:** [▶️ Tonton Tutorial di YouTube](https://youtu.be/VW8wrKlrLRk?si=9ELiSFOA4gmKgpLN)  
> **Kategori:** `VPN, PKI, Certificate Authority`

---

## 📋 Deskripsi

Lab ini membangun **Public Key Infrastructure (PKI)** dari nol di OPNsense — mencakup pembuatan Certificate Authority, Server Certificate, dan Client Certificate. PKI adalah fondasi autentikasi VPN yang memastikan setiap pihak bisa membuktikan identitasnya secara digital sebelum tunnel terenkripsi terbentuk.

---

## 🎬 Video Tutorial

[![VPN PKI Configuration - Public Key Infrastructure](https://img.youtube.com/vi/VW8wrKlrLRk/maxresdefault.jpg)](https://youtu.be/VW8wrKlrLRk?si=9ELiSFOA4gmKgpLN)

> 📺 **[Tonton di YouTube → VPN PKI Configuration: Membangun Public Key Infrastructure dari Nol](https://youtu.be/VW8wrKlrLRk?si=9ELiSFOA4gmKgpLN)**

---

## 🧠 Konsep PKI

### Apa itu Public Key Infrastructure?

**PKI (Public Key Infrastructure)** adalah sistem yang mengelola pembuatan, distribusi, dan validasi sertifikat digital. Sistem ini memastikan bahwa setiap pihak yang terlibat dalam koneksi VPN — baik server maupun client — bisa **membuktikan identitasnya secara digital**.

### Analogi dengan Sistem KTP

```
┌─────────────────────────────────────────────────────────┐
│                    PKI = Sistem KTP                     │
│                                                         │
│  CA (Certificate Authority)                             │
│  → Dinas Catatan Sipil                                  │
│    yang menerbitkan dan mengesahkan semua KTP           │
│                    │                                    │
│         ┌──────────┴──────────┐                         │
│         ▼                     ▼                         │
│  Server Certificate      Client Certificate             │
│  → KTP milik OPNsense    → KTP milik komputer fisik    │
│    (VPN server)            (yang mau konek VPN)         │
└─────────────────────────────────────────────────────────┘
```

> 💡 Ketiga komponen ini harus **ditandatangani oleh CA yang sama** agar bisa saling percaya. Kalau sertifikat server tidak berasal dari CA yang dikenal client, koneksi langsung ditolak.

---

### Kenapa Tidak Cukup Username/Password Saja?

| Metode | Kelemahan |
|--------|-----------|
| **Username + Password** | Bisa bocor, bisa ditebak, bisa di-phishing, bisa di-brute force |
| **Sertifikat Digital** | Berbentuk file kriptografis — tidak bisa ditebak atau di-brute force |
| **Certificate + Username/Password** | Standar enterprise — keamanan berlapis ✅ |

---

## 📦 Yang Akan Dibuat

```
PKI Lab VPN
├── CA (Certificate Authority)
│   └── Lab-VPN-CA              ← Otoritas tertinggi
│
├── Server Certificate
│   └── OpenVPN-Server-Cert     ← Identitas VPN server
│
├── Client Certificate
│   └── vpnuser-cert            ← Identitas pengguna VPN
│
└── User VPN
    └── vpnuser                 ← Akun untuk login VPN
```

---

## ⚙️ Langkah-Langkah Konfigurasi

### Langkah 1 — Buat Certificate Authority (CA)

```
System → Trust → Authorities → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Descriptive name** | `Lab-VPN-CA` |
| **Method** | `Create an internal Certificate Authority` |
| **Key length** | `2048` |
| **Digest Algorithm** | `SHA256` |
| **Lifetime (days)** | `3650` |
| **Country Code** | `ID` |
| **State** | `East Java` |
| **City** | *(kota kamu)* |
| **Organization** | `HomeLab` |
| **Common Name** | `Lab-VPN-CA` |

Klik **Save**.

**Verifikasi:**
```
System → Trust → Authorities
→ Pastikan Lab-VPN-CA muncul di daftar
→ Kolom Issuer: "self-signed" ← INI BENAR
```

> 💡 CA memang menandatangani dirinya sendiri karena dia adalah **otoritas tertinggi** — tidak ada pihak lain yang bisa mengesahkannya.

---

### Langkah 2 — Buat Server Certificate

```
System → Trust → Certificates → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Method** | `Create an internal Certificate` |
| **Descriptive name** | `OpenVPN-Server-Cert` |
| **Certificate Authority** | `Lab VPN-CA` |
| **Type** | `Server Certificate` |
| **Key length** | `2048` |
| **Digest Algorithm** | `SHA256` |
| **Lifetime (days)** | `3650` |
| **Common Name** | `openvpn-server` |

Klik **Save**.

**Verifikasi:**
```
Kolom Issuer harus: "Lab VPN-CA" ✅
BUKAN: "self-signed" ❌
```

> ⚠️ Kalau masih `self-signed` berarti field **Certificate Authority** terlewat saat mengisi form — hapus dan buat ulang.

---

### Langkah 3 — Buat User VPN

```
System → Access → Users → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Username** | `vpnuser` |
| **Password** | *(isi password kuat, catat di tempat aman)* |
| **Full name** | `VPN User` |

Klik **Save**.

---

### Langkah 4 — Buat Client Certificate

```
System → Trust → Certificates → klik + (Add)
```

| Field | Nilai |
|-------|-------|
| **Method** | `Create an internal Certificate` |
| **Descriptive name** | `vpnuser-cert` |
| **Certificate Authority** | `Lab VPN-CA` |
| **Type** | `Client Certificate` |
| **Key length** | `2048` |
| **Digest Algorithm** | `SHA256` |
| **Lifetime (days)** | `3650` |
| **Common Name** | `vpnuser` |

Klik **Save**.

**Verifikasi:**
```
Kolom Purpose : "id-kp-clientAuth" ✅
Kolom Issuer  : "Lab VPN-CA"       ✅
```

---

## ✅ Verifikasi Akhir

Pergi ke **System → Trust → Certificates** — pastikan daftar certificate sudah seperti ini:

| Description | Issuer | Purpose |
|-------------|--------|---------|
| Web GUI TLS certificate | self-signed | id-kp-serverAuth |
| **OpenVPN-Server-Cert** | **Lab VPN-CA** | **id-kp-serverAuth** |
| **vpnuser-cert** | **Lab VPN-CA** | **id-kp-clientAuth** |

> ✅ Dua sertifikat terakhir **harus ber-issuer Lab VPN-CA** — bukan self-signed.

---

## 🔐 Alur Kepercayaan PKI yang Terbentuk

```
Lab-VPN-CA (self-signed)
      │
      │ Menandatangani
      │
      ├──► OpenVPN-Server-Cert
      │    (id-kp-serverAuth)
      │    → OPNsense menunjukkan ini saat client konek
      │
      └──► vpnuser-cert
           (id-kp-clientAuth)
           → Client menunjukkan ini saat minta akses VPN
```

Saat koneksi VPN dibuat:
```
[1] Client menunjukkan vpnuser-cert ke server
[2] Server cek: apakah cert ini ditandatangani Lab-VPN-CA? → ✅
[3] Server menunjukkan OpenVPN-Server-Cert ke client
[4] Client cek: apakah cert ini ditandatangani Lab-VPN-CA? → ✅
[5] Keduanya saling percaya → Tunnel terenkripsi terbentuk ✅
```

---

## 🛠️ Troubleshooting

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| **Issuer self-signed di server cert** | Field Certificate Authority tidak dipilih | Hapus cert, buat ulang, pastikan CA dipilih |
| **Purpose salah di client cert** | Type dipilih Server Certificate bukan Client | Hapus cert, buat ulang dengan type Client Certificate |
| **CA tidak muncul di dropdown** | CA belum tersimpan | Kembali ke System → Trust → Authorities, pastikan CA ada |

---

## 📌 Kesimpulan

Setelah mengikuti lab ini, kamu telah berhasil:

- ✅ Membuat Certificate Authority (CA) `Lab-VPN-CA`
- ✅ Membuat Server Certificate `OpenVPN-Server-Cert` yang ditandatangani CA
- ✅ Membuat user VPN `vpnuser`
- ✅ Membuat Client Certificate `vpnuser-cert` yang ditandatangani CA
- ✅ Memverifikasi seluruh chain of trust PKI

PKI sudah terbentuk — lanjut ke lab berikutnya untuk mengkonfigurasi **OpenVPN server** menggunakan sertifikat yang sudah dibuat.

---

## 📚 Referensi

- [OPNsense PKI Documentation](https://docs.opnsense.org/manual/certificates.html)
- [OpenVPN PKI Overview](https://openvpn.net/community-resources/how-to/#pki)
- [NIST — Public Key Infrastructure](https://csrc.nist.gov/glossary/term/public_key_infrastructure)

---

*📺 Ikuti tutorialnya di [YouTube](https://youtu.be/VW8wrKlrLRk?si=9ELiSFOA4gmKgpLN) | ⭐ Star repo ini jika membantu!*
