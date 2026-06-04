# 📊 SISTEM MONITORING SERVER DAN JARINGAN SECARA REAL TIME

| Kriteria Proyek | Detail Informasi |
| :--- | :--- |
| **Kelompok** | Kelompok 5 |
| **Mata Kuliah** | Komputasi Awan / Jaringan |

---

## 🌐 Informasi Jaringan & IP Address Produksi (PROD)

Seluruh komponen didistribusikan ke dalam 3 Virtual Machine terisolasi di jaringan lokal dengan pemetaan sebagai berikut:

*   **VM 1 (Frontend - Grafana)** : [http://192.168.18.85:3000](http://192.168.18.85:3000)
*   **VM 2 (Backend - Node Exporter)** : [http://192.168.18.44:9100](http://192.168.18.44:9100)
*   **VM 3 (Database - Prometheus)** : [http://192.168.18.42:9090](http://192.168.18.42:9090)

> 💡 **Catatan Penting Infrastruktur:**
> * Seluruh layanan di VM 2 dan VM 3 dijalankan menggunakan **Docker Container** di latar belakang (*detached mode*).
> * Kebijakan Firewall (**UFW**) pada VM 2 dan VM 3 telah **dinonaktifkan** demi kelancaran jalur komunikasi matrik antar-node.

---

## 📝 1. Deskripsi Singkat Sistem

Sistem ini merupakan platform monitoring infrastruktur server dan jaringan secara *real-time* yang memisahkan beban kerja arsitektur ke dalam 3 komponen utama: **Grafana, Prometheus, dan Node Exporter**. 

Tujuan implementasi ini adalah memberikan visibilitas penuh terhadap metrik vital hardware server target secara terpusat (seperti *utilisasi CPU, sisa kapasitas RAM, aktivitas Disk I/O, serta bandwidth lalu lintas jaringan*) guna mendeteksi anomali performa secara dini pada server produksi Kelompok 5.

---

## 📐 2. Arsitektur Jaringan Cloud & Pembagian Fungsi

### 🔄 Diagram Alur Komunikasi Data Monitoring:

```text
       [ Browser Admin / Host ]
                   │
                   │ HTTP GET (Akses Dashboard)
                   ▼
       ┌───────────────────────┐
       │     VM 1 (Frontend)   │  Port : 3000
       │        Grafana        │  IP   : 192.168.18.85
       └───────────┬───────────┘
                   │
                   │ Query PromQL (Pull Metrics)
                   ▼
       ┌───────────────────────┐
       │     VM 3 (Database)   │  Port : 9090
       │   Prometheus (TSDB)   │  IP   : 192.168.18.42
       └───────────▲───────────┘
                   │
                   │ Berkala tiap 5s (Scraping Data)
                   │
       ┌───────────┴───────────┐
       │     VM 2 (Backend)    │  Port : 9100
       │     Node Exporter     │  IP   : 192.168.18.44
       └───────────────────────┘
```
### 🎛️ Pembagian Tugas Cluster VM:

1.  **VM 1 - FRONTEND MONITORING (Grafana)**
    *   **Fungsi:** Bertindak sebagai antarmuka visual (GUI). Grafana secara konstan melakukan visualisasi dengan menarik data metrik dari database waktu (*Time Series Database*) Prometheus di VM 3 untuk disajikan dalam bentuk chart interaktif.
2.  **VM 2 - BACKEND TARGET (Node Exporter)**
    *   **Fungsi:** Berperan sebagai agen sensor pengumpul metrik *raw hardware* internal dari OS Linux Kernel pada mesin target. Data tersebut diekspos secara *real-time* melalui port `9100`.
3.  **VM 3 - DATABASE SERVER (Prometheus)**
    *   **Fungsi:** Pusat penyimpanan data monitoring. Menggunakan mekanisme *scraping* (menarik data) dari target endpoint VM 2 secara berkala, lalu menyuplai data tersebut saat dipanggil oleh Grafana.

---

## 🛠️ 3. Tools & Teknologi Yang Digunakan

| Komponen | Teknologi | Keterangan / Kegunaan |
| :--- | :--- | :--- |
| **Frontend UI** | Grafana | Pembuat visualisasi grafik & manajemen dashboard monitoring |
| **TSDB Engine** | Prometheus | Database berbasis deret waktu & penarik (*scraper*) metrik |
| **Metrics Agent** | Node Exporter | Agen pengumpul metrik spesifikasi hardware Linux |
| **Orchestration** | Docker & Compose | Solusi containerization untuk standardisasi dan isolasi *environment* |
| **Operating System** | Ubuntu Server | Dasar fondasi platform OS pada ketiga Virtual Machine |

---

## 🚀 4. Panduan Menjalankan Sistem

### 🏗️ Langkah 1: Aktifkan Sensor Agen di VM 2 (`192.168.18.44`)
Masuk ke terminal VM 2, arahkan ke folder konfigurasi backend, dan hidupkan container:
```bash
cd vm2-backend-node-exporter/
sudo docker compose up -d 
```

### 🗄️ Langkah 2: Aktifkan Database di VM 3 (`192.168.18.42`)
Pastikan file `prometheus.yml` kalian sudah dikonfigurasi mengarah ke target IP VM 2, lalu nyalakan service:
```bash
cd vm3-database-prometheus/
sudo docker compose up -d 
```

### 📊 Langkah 3: Konfigurasi Tampilan Grafana di VM 1 (`192.168.18.85`)
1. Buka browser di laptop host, akses `http://192.168.18.85:3000`.
2. Masuk ke **Connections > Data Sources**, tambahkan **Prometheus** dan isi URL dengan `http://192.168.18.42:9090`.
3. Masuk ke menu **Dashboards > New > Import**, unggah berkas `dashboard-node-exporter.json` yang tersedia di repositori ini (atau gunakan ID `1860`).

## 📁 5. Struktur Folder Repositori Proyek

```text
monitoring-system/
│
├── README.md                                         # Dokumentasi utama proyek
├── monitoring server dan jaringan secara realtime.txt # Dokumentasi versi cetak teks biasa
│
├── vm1-frontend-grafana/
│   ├── docker-compose.yml                            # Konfigurasi container engine Grafana
│   └── dashboard-node-exporter.json                  # Berkas blueprint layout ekspor grafik
│
├── vm2-backend-node-exporter/
│   └── docker-compose.yml                            # Berkas peluncur container agen Node Exporter
│
└── vm3-database-prometheus/
    ├── docker-compose.yml                            # Berkas peluncur database Prometheus
    └── prometheus.yml                                # Aturan target penembakan metrik ke VM 2
```
	
## 🔄 6. Alur Kerja Monitoring Real-Time

1. **Pengumpulan Metrik:** Node Exporter yang berjalan sebagai container di **VM 2** secara aktif membaca status *resource* hardware langsung dari kernel sistem Linux (CPU, memori, penyimpanan, dan jaringan) setiap detik.
2. **Scraping (Penarikan Data):** Layanan Prometheus di **VM 3** bertindak aktif melakukan perintah *pull request* (scraping) ke endpoint agen di alamat `http://192.168.18.44:9100/metrics` secara berkala (default tiap 5-15 detik) lalu menyimpannya ke dalam database internal berbasis waktu (*Time Series Database*).
3. **Visualisasi Data:** Ketika administrator membuka browser dan mengakses Grafana di **VM 1**, panel dashboard otomatis mengirimkan kueri data menggunakan bahasa *PromQL* ke Prometheus di **VM 3** (port `9090`). Grafana kemudian langsung merender data tersebut menjadi grafik interaktif yang diperbarui otomatis di layar tanpa perlu melakukan *refresh* halaman penuh.