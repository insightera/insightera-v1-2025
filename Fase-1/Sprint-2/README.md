# Sprint 2 – Integrasi HDFS dan Azure Data Lake (Hybrid Mount)

Sprint 2 berfokus pada proses **integrasi antara Hadoop Distributed File System (HDFS)** di cluster INSIGHTERA dengan **Azure Data Lake Storage Gen2 (ADLS Gen2)** untuk membentuk arsitektur **Hybrid Data Lakehouse**.  
Tujuan utama sprint ini adalah memungkinkan sistem Hadoop–Spark INSIGHTERA melakukan akses baca-tulis langsung ke penyimpanan cloud Azure menggunakan protokol `abfs://`, sehingga data dapat dikelola secara terdistribusi antara lingkungan lokal dan cloud.

---

## Tujuan Sprint 2

1. Menyiapkan **Azure Data Lake Storage Gen2** dengan struktur direktori berbasis *Medallion Architecture* (Bronze, Silver, Gold).  
2. Menyusun **15 domain data mart unit kerja ITERA** di seluruh lapisan penyimpanan.  
3. Membangun **Service Principal** di Azure AD dan mengonfigurasikan autentikasi **OAuth2 Client Credential Flow** untuk Hadoop.  
4. Menghubungkan Hadoop Cluster dengan ADLS melalui driver `abfs://` dan mengimplementasikan *hybrid mount*.  
5. Melakukan **pengujian sinkronisasi dua arah** antara HDFS ↔ ADLS, mengukur performa dan memastikan konsistensi data.

---

## Struktur Folder Sprint 2

| File | Deskripsi |
|------|------------|
| **sprint-2-1.md** | Sprint 2 Bagian 1 – Pembuatan Storage Account ADLS Gen2, tiga container (bronze, silver, gold), serta 15 folder domain data mart INSIGHTERA. |
| **sprint-2-2.md** | Sprint 2 Bagian 2 – Pembuatan Service Principal di Azure AD dan konfigurasi autentikasi OAuth2 untuk koneksi Hadoop–ADLS. |
| **sprint-2-3.md** | Sprint 2 Bagian 3 – Pengaturan Hadoop (core-site.xml dan hdfs-site.xml) agar mendukung koneksi `abfs://` serta pembuatan hybrid mount antar sistem. |
| **sprint-2-4.md** | Sprint 2 Bagian 4 – Pengujian koneksi dua arah, sinkronisasi file, throughput transfer, dan validasi konsistensi data HDFS–ADLS. |

---

## Ringkasan Tahapan Sprint 2

### Bagian 1 – Persiapan Azure Data Lake Storage Gen2  
- Membuat Storage Account `insighteradata` dengan **Hierarchical Namespace** aktif.  
- Membuat tiga container: `bronze`, `silver`, `gold`.  
- Menyusun 15 folder sesuai domain data mart INSIGHTERA:  
  `akademik`, `nonakademik`, `kemahasiswaan`, `lppm`, `lpmp`, `keuangan`, `kepegawaian`, `keamanansiber`, `satudata`, `kebunraya`, `fsains`, `fti`, `ftik`, `sarpras`, `k3l`.  
- Struktur direktori dibuat seragam di ketiga container.

### Bagian 2 – Pembuatan Service Principal dan Autentikasi OAuth2  
- Membuat **Service Principal `sp-insightera`** di Azure Active Directory.  
- Memberikan peran **Storage Blob Data Contributor** pada resource group `RG-Datalakehouse-Insightera`.  
- Mengonfigurasi file `core-site.xml` agar Hadoop dapat melakukan autentikasi otomatis via OAuth2.  
- Uji koneksi `hdfs dfs -ls abfs://bronze@insighteradata.dfs.core.windows.net/` berhasil.

### Bagian 3 – Konfigurasi Hadoop dan Implementasi Hybrid Mount  
- Menyesuaikan `core-site.xml` dan `hdfs-site.xml` di seluruh node cluster.  
- Menghubungkan direktori HDFS `/insightera/bronze`, `/insightera/silver`, dan `/insightera/gold` dengan container ADLS menggunakan `distcp`.  
- Uji tulis dan baca file melalui protokol `abfs://` berhasil.  
- Sistem dapat melakukan *bidirectional read/write* antar HDFS ↔ ADLS.

### Bagian 4 – Pengujian Sinkronisasi dan Validasi Kinerja  
- Uji transfer data arah **HDFS → ADLS** dan **ADLS → HDFS** menggunakan `hadoop distcp`.  
- Rata-rata throughput transfer: **20–25 MB/s**.  
- Latency baca file kecil dari ADLS: **0.25 detik**.  
- MD5 checksum file identik di kedua sisi (tidak ada korupsi data).  
- Semua 15 domain data mart tersinkronisasi dan tervalidasi.  

---

## Struktur Hybrid Data Lakehouse yang Terbentuk

```

HDFS (/opt/hadoop_data/)
└── insightera/
├── bronze/ → abfs://bronze@insighteradata.dfs.core.windows.net/
├── silver/ → abfs://silver@insighteradata.dfs.core.windows.net/
└── gold/   → abfs://gold@insighteradata.dfs.core.windows.net/

ADLS Gen2 (insighteradata.dfs.core.windows.net)
├── bronze/
│   ├── akademik/
│   ├── keuangan/
│   └── ...
├── silver/
│   ├── akademik/
│   ├── keuangan/
│   └── ...
└── gold/
├── akademik/
├── keuangan/
└── ...

```

---

## Hasil Akhir Sprint 2

| Komponen | Status | Keterangan |
|-----------|---------|------------|
| Storage Account ADLS Gen2 | Aktif | Hierarchical Namespace diaktifkan |
| Container Bronze/Silver/Gold | Dibuat | Struktur Medallion lengkap |
| 15 Unit Data Mart | Dibentuk | Tersedia di setiap container |
| Service Principal | Berhasil dibuat | sp-insightera dengan role Storage Contributor |
| Autentikasi OAuth2 | Berfungsi | Hadoop dapat login otomatis |
| Protokol abfs:// | Aktif | Mendukung koneksi langsung Hadoop–ADLS |
| Sinkronisasi HDFS ↔ ADLS | Berhasil dua arah | distcp dan upload langsung berfungsi |
| Validasi Konsistensi | Lulus | MD5 checksum identik di kedua sisi |

---

## Kesimpulan Sprint 2

Dengan selesainya Sprint 2, arsitektur **Hybrid Data Lakehouse INSIGHTERA** kini telah terbentuk secara penuh.  
Cluster Hadoop–Spark dapat mengakses dan menyimpan data langsung ke Azure Data Lake Storage Gen2 menggunakan driver `abfs://` dengan autentikasi aman berbasis OAuth2.  
Struktur 15 domain unit kerja telah tersedia di tiga lapisan Medallion (Bronze, Silver, Gold) dan siap digunakan untuk proses ingest, transformasi, dan federasi pada fase berikutnya.

---

## Arah Pengembangan Berikutnya

Tahapan selanjutnya akan dilanjutkan pada **Sprint 3**, yang berfokus pada:
1. Pembuatan dan konfigurasi **Hive Metastore berbasis PostgreSQL**.  
2. Integrasi metadata antara Hive, Hadoop, dan ADLS.  
3. Persiapan federasi awal untuk tiap **data mart domain** di INSIGHTERA.  
4. Validasi query SQL dan akses terdistribusi antar layer (Bronze → Silver → Gold).  

---

**Dokumentasi:**  
Proyek INSIGHTERA – Fase I: Infrastruktur dan Data Lakehouse (Hybrid Core)  
Sprint 2: Integrasi HDFS dan Azure Data Lake (Hybrid Mount)  
Disusun oleh: Ardika Satria, M.Si – Fakultas Sains, Institut Teknologi Sumatera  
Tahun: 2025