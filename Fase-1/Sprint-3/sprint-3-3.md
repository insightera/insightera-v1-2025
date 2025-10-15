### Sprint 3 – Bagian 3

**Pembuatan Skema dan Database Hive untuk 15 Data Mart INSIGHTERA**

Bagian ini merupakan langkah penting dalam membangun **lapisan metadata terstruktur** untuk sistem INSIGHTERA. Setelah Apache Hive 4.1.0 berhasil terintegrasi dengan HDFS dan ADLS Gen2, tahapan ini berfokus pada pembuatan **database Hive (schema)** untuk setiap **15 unit kerja atau domain data mart** di ITERA.

Masing-masing database Hive akan merepresentasikan area fungsional (domain) yang berbeda di dalam *Hybrid Data Lakehouse*, dan berisi tabel-tabel eksternal yang menunjuk ke direktori data yang tersimpan di **ADLS Gen2** (Bronze, Silver, Gold).

Dengan demikian, Hive Metastore akan menjadi pusat metadata federatif yang menghubungkan seluruh domain data mart ITERA.

---

#### 1. Tujuan Bagian Ini

1. Membuat 15 database Hive untuk setiap unit kerja INSIGHTERA.
2. Menentukan lokasi database di ADLS menggunakan protokol `abfs://`.
3. Membuat tabel eksternal untuk masing-masing layer (Bronze, Silver, Gold).
4. Menguji query lintas database dan memastikan metadata tersimpan di PostgreSQL Metastore.
5. Memastikan struktur schema seragam dan dapat diakses oleh SparkSQL maupun HiveServer2.

---

#### 2. Struktur Data Mart INSIGHTERA

| No | Unit / Domain                        | Nama Database Hive | Lokasi di ADLS                                                    |
| -- | ------------------------------------ | ------------------ | ----------------------------------------------------------------- |
| 1  | Akademik                             | `akademik_db`      | `abfs://silver@insighteradata.dfs.core.windows.net/akademik`      |
| 2  | Non Akademik                         | `nonakademik_db`   | `abfs://silver@insighteradata.dfs.core.windows.net/nonakademik`   |
| 3  | Kemahasiswaan                        | `kemahasiswaan_db` | `abfs://silver@insighteradata.dfs.core.windows.net/kemahasiswaan` |
| 4  | LPPM                                 | `lppm_db`          | `abfs://silver@insighteradata.dfs.core.windows.net/lppm`          |
| 5  | LPMPP                                | `lpmp_db`          | `abfs://silver@insighteradata.dfs.core.windows.net/lpmp`          |
| 6  | Keuangan                             | `keuangan_db`      | `abfs://silver@insighteradata.dfs.core.windows.net/keuangan`      |
| 7  | Kepegawaian                          | `kepegawaian_db`   | `abfs://silver@insighteradata.dfs.core.windows.net/kepegawaian`   |
| 8  | Keamanan Siber                       | `keamanansiber_db` | `abfs://silver@insighteradata.dfs.core.windows.net/keamanansiber` |
| 9  | Satu Data ITERA                      | `satudata_db`      | `abfs://silver@insighteradata.dfs.core.windows.net/satudata`      |
| 10 | Kebun Raya ITERA                     | `kebunraya_db`     | `abfs://silver@insighteradata.dfs.core.windows.net/kebunraya`     |
| 11 | Fakultas Sains                       | `fsains_db`        | `abfs://silver@insighteradata.dfs.core.windows.net/fsains`        |
| 12 | Fakultas Teknologi Industri          | `fti_db`           | `abfs://silver@insighteradata.dfs.core.windows.net/fti`           |
| 13 | Fakultas Infrastruktur & Kewilayahan | `fik_db`           | `abfs://silver@insighteradata.dfs.core.windows.net/fik`           |
| 14 | Sarana & Prasarana                   | `sarpras_db`       | `abfs://silver@insighteradata.dfs.core.windows.net/sarpras`       |
| 15 | K3L                                  | `k3l_db`           | `abfs://silver@insighteradata.dfs.core.windows.net/k3l`           |

---

#### 3. Pembuatan Database Hive

Masuk ke Hive CLI dari VM-Master:

```bash
hive
```

Lalu jalankan skrip berikut secara berurutan untuk membuat 15 database Hive:

```sql
CREATE DATABASE akademik_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/akademik';
CREATE DATABASE nonakademik_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/nonakademik';
CREATE DATABASE kemahasiswaan_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/kemahasiswaan';
CREATE DATABASE lppm_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/lppm';
CREATE DATABASE lpmp_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/lpmp';
CREATE DATABASE keuangan_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/keuangan';
CREATE DATABASE kepegawaian_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/kepegawaian';
CREATE DATABASE keamanansiber_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/keamanansiber';
CREATE DATABASE satudata_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/satudata';
CREATE DATABASE kebunraya_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/kebunraya';
CREATE DATABASE fsains_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/fsains';
CREATE DATABASE fti_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/fti';
CREATE DATABASE fik_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/fik';
CREATE DATABASE sarpras_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/sarpras';
CREATE DATABASE k3l_db LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/k3l';
```

Verifikasi:

```sql
SHOW DATABASES;
```

Jika seluruh 15 database muncul, maka Metastore PostgreSQL telah menyimpan seluruh metadata domain.

---

#### 4. Pembuatan Tabel Eksternal pada Tiap Database

Contoh implementasi untuk satu domain (misal: `akademik_db`):

```sql
USE akademik_db;

CREATE EXTERNAL TABLE mahasiswa (
  npm STRING,
  nama STRING,
  prodi STRING,
  angkatan INT,
  ipk FLOAT
)
STORED AS PARQUET
LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/akademik/mahasiswa';

CREATE EXTERNAL TABLE dosen (
  nidn STRING,
  nama STRING,
  jabatan STRING,
  fakultas STRING
)
STORED AS PARQUET
LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/akademik/dosen';
```

Gunakan pola yang sama untuk setiap database dengan nama tabel sesuai domain masing-masing.
Setiap tabel eksternal menunjuk ke direktori data pada container ADLS Gen2.

---

#### 5. Uji Query dan Validasi Metadata

Coba query sederhana:

```sql
SELECT * FROM akademik_db.mahasiswa;
```

Jika data belum ada, tambahkan contoh data menggunakan perintah `INSERT`:

```sql
INSERT INTO akademik_db.mahasiswa VALUES 
('120140001', 'Dita Sari', 'Teknik Informatika', 2020, 3.75),
('120140002', 'Budi Raharjo', 'Fisika', 2019, 3.56);
```

Lalu cek isi direktori di ADLS:

```
/silver/akademik/mahasiswa/
```

Akan muncul file parquet hasil penyimpanan Hive.

---

#### 6. Validasi Penyimpanan Metadata di PostgreSQL

Jalankan perintah di VM Management:

```bash
sudo -u postgres psql -d hive_metastore -c "SELECT COUNT(*) FROM TBLS;"
sudo -u postgres psql -d hive_metastore -c "SELECT DB_ID, NAME FROM DBS;"
```

Pastikan jumlah database (`DBS`) = 15 dan setiap database memiliki beberapa tabel (`TBLS`).

---

#### 7. Pengujian Akses dari SparkSQL

Jalankan perintah dari SparkSQL di VM Master:

```bash
spark-sql --master yarn --conf spark.sql.catalogImplementation=hive -e "SHOW DATABASES;"
spark-sql --master yarn --conf spark.sql.catalogImplementation=hive -e "SELECT * FROM akademik_db.mahasiswa;"
```

Jika hasil sama dengan Hive CLI, integrasi Metastore antar komponen sudah berhasil.

---

#### 8. Struktur Metadata Hive INSIGHTERA

Hive Metastore kini memiliki struktur metadata berikut:

```
Hive Metastore (PostgreSQL)
├── akademik_db
│   ├── mahasiswa
│   └── dosen
├── keuangan_db
│   ├── transaksi
│   └── anggaran
├── kemahasiswaan_db
│   ├── kegiatan
│   └── organisasi
...
└── k3l_db
    ├── insiden
    └── laporan
```

---

#### 9. Hasil Akhir Sprint 3 Bagian 3

| Komponen              | Status    | Keterangan                    |
| --------------------- | --------- | ----------------------------- |
| Hive Databases        | 15 dibuat | Sesuai domain INSIGHTERA      |
| External Tables       | Dibuat    | Lokasi di ADLS (silver layer) |
| Metadata Storage      | Aktif     | PostgreSQL metastore          |
| Query Hive & SparkSQL | Berhasil  | Terhubung ke sumber yang sama |
| Struktur Federasi     | Terbentuk | Per domain data mart          |

---

#### 10. Kesimpulan

Bagian ini berhasil membangun lapisan metadata terdistribusi untuk seluruh 15 data mart INSIGHTERA melalui **Hive 4.1.0** dengan backend PostgreSQL.
Setiap domain kini memiliki database Hive yang menunjuk langsung ke penyimpanan hybrid di ADLS Gen2.
Konfigurasi ini menjadi pondasi bagi sistem federasi dan analitik terintegrasi INSIGHTERA.

Langkah berikutnya akan dilanjutkan pada **Sprint 3 – Bagian 4: Pengujian Query Federasi dan Validasi Integrasi Metadata antar Domain dan Layer.**
