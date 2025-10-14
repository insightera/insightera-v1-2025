# Fase I – Infrastruktur dan Data Lakehouse (Hybrid Core)

Dokumentasi ini mencakup seluruh proses **pembangunan infrastruktur hybrid INSIGHTERA** menggunakan pendekatan *Infrastructure as a Service (IaaS)* di Azure, serta konfigurasi Hadoop dan Spark sebagai fondasi utama sistem Big Data ITERA.

Fase ini merupakan fondasi dari keseluruhan arsitektur INSIGHTERA, yang berfungsi membangun lingkungan komputasi terdistribusi berbasis Hadoop–Spark dan menyiapkan konektivitas ke penyimpanan cloud (Azure Data Lake) sebagai bagian dari arsitektur *Hybrid Data Lakehouse*.

---

## Tujuan Fase I
1. Menyediakan lingkungan komputasi hybrid berbasis empat Virtual Machine di Azure (1 Master, 1 Management, 2 Worker).
2. Menginstal dan mengonfigurasi Hadoop 3.4.2 dan Spark 4.0.1 sebagai mesin komputasi terdistribusi.
3. Menguji performa, replikasi, dan stabilitas cluster.
4. Menyiapkan sistem agar siap diintegrasikan dengan Azure Data Lake Storage Gen2 pada fase berikutnya.

---

## Struktur Folder
Semua bagian Sprint 1 disusun secara berurutan dan dapat ditemukan di dalam folder ini:

| File | Deskripsi |
|------|------------|
| **sprint-1-1.md** | Sprint 1 Bagian 1 – Instalasi Azure CLI, login, pembuatan Resource Group, Virtual Network, dan Subnet sesuai topologi INSIGHTERA. |
| **sprint-1-2.md** | Sprint 1 Bagian 2 – Pembuatan empat VM Ubuntu 24.04 (Master, Management, Worker1, Worker2) dengan konfigurasi jaringan statis. |
| **sprint-1-3a.md** | Sprint 1 Bagian 3a – Instalasi dan konfigurasi Hadoop 3.4.2 pada seluruh node (Master dan Worker), termasuk setup HDFS dan YARN. |
| **sprint-1-3b.md** | Sprint 1 Bagian 3b – Instalasi dan konfigurasi Apache Spark 4.0.1, integrasi dengan Hadoop, uji Spark Master–Worker, dan SparkPi test. |
| **sprint-1-4.md** | Sprint 1 Bagian 4 – Pengujian kinerja dan validasi cluster Hadoop–Spark, meliputi replikasi data, throughput HDFS, dan uji Spark di YARN. |

---

## Ringkasan Tahapan Sprint 1

### Sprint 1 – Bagian 1  
**Persiapan Azure CLI dan Konfigurasi Dasar**  
Menyiapkan lingkungan kerja Azure di macOS/Linux. Membuat Resource Group `RG-Datalakehouse-Insightera`, VNet `VNet-Datalakehouse-Insightera`, dan empat subnet:  
- `Subnet-Compute` (10.0.1.0/24) untuk node komputasi  
- `Subnet-Storage` (10.0.2.0/24) untuk integrasi ADLS  
- `Subnet-Management` (10.0.3.0/24) untuk layanan administratif  
- `Subnet-Bastion` (10.0.4.0/24) untuk akses aman administrator  

### Sprint 1 – Bagian 2  
**Pembuatan Virtual Machine Cluster**  
Deploy empat VM Ubuntu 24.04 LTS (2 vCPU / 4 GB / 64 GB) pada subnet Compute.  
Konfigurasi SSH keypair, IP statis, dan host mapping antar node.  
Node yang dihasilkan:
- VM-Master  
- VM-Management  
- VM-Worker1  
- VM-Worker2  

### Sprint 1 – Bagian 3a  
**Instalasi dan Konfigurasi Hadoop 3.4.2**  
Pemasangan Hadoop di semua node.  
Konfigurasi `core-site.xml`, `hdfs-site.xml`, `yarn-site.xml`, dan `workers`.  
HDFS diformat, cluster dijalankan, dan diuji dengan perintah dasar (upload–read–replication).  
Hasil: HDFS aktif dengan dua DataNode dan faktor replikasi = 2.

### Sprint 1 – Bagian 3b  
**Instalasi dan Konfigurasi Apache Spark 4.0.1**  
Pemasangan Spark di semua node dengan integrasi penuh ke Hadoop.  
Konfigurasi `spark-env.sh`, `workers`, dan direktori log di HDFS.  
Cluster diuji menggunakan `spark-submit` untuk contoh `SparkPi` dan mode YARN.  
Hasil: Spark Master dan dua Worker aktif, History Server berjalan di port 18080.

### Sprint 1 – Bagian 4  
**Pengujian Kinerja dan Validasi Cluster**  
Pengujian throughput HDFS menggunakan `TestDFSIO`, uji job SparkPi, serta validasi replikasi data dan komunikasi antar node.  
Hasil rata-rata throughput 95 MB/s dan waktu eksekusi SparkPi ±18 detik untuk 100 partisi.

---

## Hasil Akhir Fase I

| Komponen | Status | Keterangan |
|-----------|---------|------------|
| HDFS | Aktif | Replikasi berjalan pada dua DataNode |
| YARN | Aktif | Resource Manager dan Node Manager terhubung |
| Spark | Aktif | Master, Worker, dan History Server berjalan |
| Integrasi | Siap | Siap dihubungkan ke Azure Data Lake |
| Performa | Stabil | Throughput ~95 MB/s, latency rendah |

---

## Arah Pengembangan Selanjutnya

Tahapan berikutnya akan dilakukan pada **Fase II (Sprint 2–5)**, yang mencakup:

1. Integrasi HDFS dengan **Azure Data Lake Storage Gen2 (Hybrid Mount)**  
2. Penyiapan **Hive Metastore** berbasis PostgreSQL  
3. Implementasi **Airflow DAG Ingestion** untuk pipeline ETL  
4. Integrasi dengan **Azure Synapse Analytics** dan **Power BI**

---

**Catatan:**  
Fase I ini menjadi dasar dari seluruh ekosistem INSIGHTERA. Semua konfigurasi yang dihasilkan disarankan untuk disimpan di repositori `infrastructure/config/insightera/` agar dapat direplikasi dengan mudah pada pengembangan berikutnya.

---

**Dokumentasi:**  
Proyek INSIGHTERA – Pengembangan Sistem Big Data ITERA  
Fase I: Infrastruktur dan Data Lakehouse (Hybrid Core)  
Disusun oleh: Ardika Satria, M.Si – Fakultas Sains, ITERA  
Tahun: 2025
