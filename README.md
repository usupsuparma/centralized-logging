# Centralized Logging Server (Grafana Loki & Grafana Dashboard)

Aplikasi server monitoring log terpusat (*Centralized Logging Server*) berbasis **Grafana Loki** dan **Grafana**.

Server ini bertindak sebagai **pusat penerimaan, pengindeksan, penyimpanan, dan visualisasi log** dari seluruh server aplikasi (seperti **HRIS Gold Chick** dan aplikasi/microservice lain di masa mendatang).

> 💡 **Catatan Arsitektur:**  
> Server ini **hanya menjalankan Loki dan Grafana**.  
> Agen pengumpul log (**Grafana Alloy**) berjalan di masing-masing Server Aplikasi (contoh: pada repository `hris-goldchick` di folder `services/grafana-alloy/`).

---

## 🏛️ Arsitektur Sistem Terdistribusi

```text
┌─────────────────────────────────────────┐          ┌─────────────────────────────────────────┐
│        SERVER APLIKASI (App Server)     │          │      SERVER MONITORING (Repo Ini)       │
│                                         │          │                                         │
│  [ Aplikasi HRIS Gold Chick ]           │          │  [ Grafana Loki ]                       │
│          │                              │          │         ▲  (Port 3100)                  │
│          ▼ (Tulis Log NDJSON)           │          │         │                               │
│    storage/logs/*.log                   │          │         │ (Menerima log stream)         │
│          │                              │          │         │                               │
│          ▼                              │          │         │                               │
│  [ Grafana Alloy (Agent Lokal) ]        │          │         │                               │
│          │                              │          │         │                               │
│          └──────────────────────────────┼──────────┘         │                               │
│               Kirim via HTTP/HTTPS      │                    │                               │
│               POST /loki/api/v1/push    │                    ▼                               │
│                                         │          [ Grafana Dashboard (Port 3000) ]         │
└─────────────────────────────────────────┘          └─────────────────────────────────────────┘
```

---

## 📁 Struktur Direktori Server Monitoring

```text
centralized-logging/
├── docker-compose.yml              # Orkestrasi container Loki (Port 3100) dan Grafana (Port 3000)
├── .env.example                    # Template environment (port & kredensial)
├── .gitignore                      # Mengabaikan data lokal & file .env
├── README.md                       # Dokumentasi arsitektur dan operasional
├── loki/
│   └── loki-config.yaml            # Konfigurasi engine Loki, TSDB index, dan retensi 30 hari
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── loki-datasource.yaml   # Auto-koneksi Grafana ke Loki + link RequestID
│   │   └── dashboards/
│   │       └── dashboards.yaml        # Auto-import dashboard monitoring ke Grafana
│   └── dashboards/
│       └── hris-monitoring-dashboard.json # Dashboard visual monitoring siap pakai
└── nginx/
    └── loki-reverse-proxy.conf     # Template reverse proxy Nginx (SSL/TLS & Basic Auth)
```

---

## 🚀 Panduan Memulai Cepat (Quick Start)

### 1. Salin Konfigurasi Environment
Salin file `.env.example` menjadi `.env`:
```bash
cp .env.example .env
```

Sesuaikan port dan password admin Grafana di `.env`:
```dotenv
GRAFANA_PORT=3000
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin
LOKI_PORT=3100
```

### 2. Jalankan Server Menggunakan Docker Compose
```bash
docker compose up -d
```

### 3. Akses Antarmuka Web
| Layanan | URL | Kredensial Default | Fungsi |
|---|---|---|---|
| **Grafana Dashboard** | `http://IP_SERVER_MONITORING:3000` | User: `admin` <br> Password: `admin` | Visualisasi metrik, ringkasan error, dan log viewer |
| **Grafana Loki API** | `http://IP_SERVER_MONITORING:3100/ready` | N/A | Status kesiapan engine penyimpanan log |

---

## 📊 Dashboard Monitoring Bawaan

Begitu Grafana dibuka, dashboard **`HRIS Gold Chick - Centralized Log Monitoring`** langsung tersedia di folder **`Gold Chick Applications`** dengan fitur:

1. **Executive Stat Cards**:
   - Total Log Events
   - Total Error & Critical Events
   - Total Warnings
   - Normal Business Activity Count (INFO)
2. **Tren Waktu Log**: Grafik baris bertingkat (*stacked timeseries*) berdasarkan tingkat urgensi log (`INFO`, `WARNING`, `ERROR`, `CRITICAL`).
3. **Modul Distribution**: Donut chart aktivitas modul HRIS (`auth`, `attendance`, `payroll`, `leave`, `reimbursement`, dll).
4. **Top 10 Business Actions**: Bar gauge aksi bisnis teratas (misal `auth.login`, `attendance.clock_in`, `payroll.generate`).
5. **Real-time Log Stream**: Live log viewer yang mendukung:
   - Filter dropdown berdasarkan **Aplikasi** (`app`).
   - Filter multi-pilih berdasarkan **Level** (`INFO`, `WARNING`, `ERROR`).
   - Filter multi-pilih berdasarkan **Modul**.
   - Kolom pencarian teks bebas atau UUID `request_id`.
   - Fitur klik pada `RequestID` untuk melihat seluruh alur log dalam satu request yang sama.

---

## 🔎 Cheatsheet Query LogQL Populer

Di menu **Explore** Grafana (`http://IP_SERVER_MONITORING:3000/explore`), pilih datasource **Loki**, lalu gunakan query berikut:

#### 1. Menampilkan seluruh log error HRIS:
```logql
{app="hris-goldchick", level=~"ERROR|CRITICAL"}
```

#### 2. Melacak seluruh log dari satu Request ID (korelasi request pengguna):
```logql
{app="hris-goldchick"} | json | request_id = "9b1e9c20-a7d1-4467-8e6f-123456789abc"
```

#### 3. Memantau aktivitas modul Payroll:
```logql
{app="hris-goldchick", module="payroll"} | json
```

#### 4. Menghitung laju error per menit:
```logql
sum(rate({app="hris-goldchick", level="ERROR"}[1m]))
```

#### 5. Memantau kegagalan login:
```logql
{app="hris-goldchick", action="auth.login", level="WARNING"} | json
```

---

## 🛡️ Keamanan & Rekomendasi di Production

1. **Ubah Kredensial Grafana Default**: Ubah `GRAFANA_ADMIN_PASSWORD` pada `.env` dengan password yang aman.
2. **Reverse Proxy (Nginx / Cloudflare)**: Pasang Nginx reverse proxy dengan HTTPS (SSL/TLS) di depan port 3000 dan port 3100 jika Server Monitoring diakses dari internet publik. Template konfigurasi tersedia di `nginx/loki-reverse-proxy.conf`.
3. **Retensi Penyimpanan**: Pengaturan default di `loki/loki-config.yaml` adalah **30 hari** (`retention_period: 720h`). Anda dapat memperpanjang atau memperpendeknya sesuai kapasitas disk server.
