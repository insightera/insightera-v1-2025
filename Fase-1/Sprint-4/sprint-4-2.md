### Sprint 4 – Bagian 2

**Konfigurasi Connection, Variable, dan Environment untuk Pipeline Hybrid INSIGHTERA**

Bagian ini melanjutkan tahapan sebelumnya setelah **Apache Airflow 3.1.0** berhasil terinstal dan dikonfigurasi di VM Management.
Tujuan dari bagian ini adalah menyiapkan seluruh **connection, variable, dan environment** yang dibutuhkan agar Airflow dapat berkomunikasi dengan komponen utama INSIGHTERA: **Hadoop (HDFS)**, **Spark**, **Hive Metastore**, dan **Azure Data Lake Gen2 (ADLS)**.
Tahapan ini akan menjadi fondasi untuk pembangunan **DAG ETL Hybrid** pada bagian selanjutnya.

---

#### 1. Tujuan Bagian Ini

1. Menambahkan koneksi (Airflow Connections) untuk Hadoop, Spark, Hive, dan ADLS.
2. Menentukan variabel global Airflow (Airflow Variables) untuk jalur data Bronze–Silver–Gold.
3. Menyimpan credential OAuth2 untuk ADLS secara aman melalui `.env` dan Airflow Secret backend.
4. Mengonfigurasi *environment path* dan *connection testing* untuk memastikan seluruh hook Airflow berfungsi.
5. Menyiapkan struktur proyek agar siap digunakan untuk pipeline ETL Hybrid INSIGHTERA.

---

#### 2. Struktur Proyek Airflow Setelah Sprint 4 Bagian 1

Direktori Airflow di VM Management setelah instalasi:

```
/opt/airflow/
├── dags/
│   ├── etl_bronze_to_silver.py
│   ├── etl_silver_to_gold.py
│   └── validation_pipeline.py
├── logs/
├── plugins/
├── venv/
├── airflow.cfg
└── .env
```

File `.env` akan digunakan untuk menyimpan **credential rahasia** (misalnya OAuth2 client secret ADLS) yang tidak disimpan langsung di `airflow.cfg`.

---

#### 3. Menambahkan Koneksi di Airflow UI

Masuk ke **Airflow Web UI**
`http://<Public-IP-VM-Management>:8080`
Login sebagai `admin`.

Buka menu **Admin → Connections → + Add Connection**, lalu tambahkan koneksi berikut:

| Connection ID         | Conn Type       | Host                                | Port | Login/User | Password      | Extra (JSON / URI)                                  |
| --------------------- | --------------- | ----------------------------------- | ---- | ---------- | ------------- | --------------------------------------------------- |
| `hdfs_conn`           | HDFS            | master                              | 9000 | hadoop     |               | `{ "user": "hadoop" }`                              |
| `spark_conn`          | Spark           | master                              | 7077 |            |               | `{ "deploy-mode": "cluster" }`                      |
| `hive_metastore_conn` | Hive Metastore  | master                              | 9083 | hive_user  | hive_password | `{ "schema": "hive_metastore" }`                    |
| `adls_conn`           | Azure Data Lake | insighteradata.dfs.core.windows.net | 443  | client_id  | client_secret | `{ "tenant": "<tenant_id>", "auth_type": "OAuth" }` |

**Catatan:**

* Connection `adls_conn` digunakan oleh Airflow Hook (misal `WasbHook` atau `AzureDataLakeHook`).
* Connection `hdfs_conn` akan digunakan untuk mengakses HDFS dan memindahkan hasil transformasi sebelum diunggah ke ADLS.

---

#### 4. Menambahkan Airflow Variables

Masuk ke menu **Admin → Variables → + Add a new record**, lalu isi variabel global berikut:

| Key            | Value                                                | Deskripsi                   |
| -------------- | ---------------------------------------------------- | --------------------------- |
| `BRONZE_PATH`  | `abfs://bronze@insighteradata.dfs.core.windows.net/` | Lokasi layer Bronze di ADLS |
| `SILVER_PATH`  | `abfs://silver@insighteradata.dfs.core.windows.net/` | Lokasi layer Silver di ADLS |
| `GOLD_PATH`    | `abfs://gold@insighteradata.dfs.core.windows.net/`   | Lokasi layer Gold di ADLS   |
| `HDFS_PATH`    | `hdfs://master:9000/user/insightera/`                | Lokasi kerja lokal di HDFS  |
| `SPARK_HOME`   | `/opt/spark`                                         | Lokasi instalasi Spark      |
| `HADOOP_HOME`  | `/opt/hadoop`                                        | Lokasi instalasi Hadoop     |
| `HIVE_HOME`    | `/opt/hive`                                          | Lokasi instalasi Hive       |
| `ADLS_ACCOUNT` | `insighteradata`                                     | Nama Storage Account ADLS   |
| `PROJECT_NAME` | `INSIGHTERA`                                         | Nama proyek utama           |
| `REGION`       | `southeastasia`                                      | Lokasi resource Azure       |
| `ENV`          | `production`                                         | Mode environment pipeline   |

**Tujuan:**
Semua variable ini akan dipanggil oleh DAG menggunakan:

```python
from airflow.models import Variable
bronze_path = Variable.get("BRONZE_PATH")
```

---

#### 5. Menyimpan Credential Rahasia di `.env`

Untuk menjaga keamanan koneksi ke ADLS dan PostgreSQL Hive Metastore, buat file `.env` di `/opt/airflow/`:

```bash
nano /opt/airflow/.env
```

Isi file dengan format:

```
# Azure ADLS OAuth2 Credentials
ADLS_TENANT_ID=<tenant_id>
ADLS_CLIENT_ID=<client_id>
ADLS_CLIENT_SECRET=<client_secret>

# PostgreSQL Hive Metastore
HIVE_DB_USER=hive_user
HIVE_DB_PASS=hive_password
HIVE_DB_HOST=master
HIVE_DB_PORT=5432

# Spark Cluster
SPARK_MASTER_URL=spark://master:7077

# Environment
AIRFLOW_ENV=production
```

Atur permission agar tidak bisa diakses publik:

```bash
chmod 600 /opt/airflow/.env
```

---

#### 6. Konfigurasi Environment Variable Airflow

Edit file environment service agar Airflow otomatis memuat `.env`:

```bash
nano ~/.bashrc
```

Tambahkan:

```
export AIRFLOW_HOME=/opt/airflow
set -a
source /opt/airflow/.env
set +a
```

Terapkan perubahan:

```bash
source ~/.bashrc
```

Verifikasi bahwa environment termuat:

```bash
echo $ADLS_CLIENT_ID
```

---

#### 7. Pengujian Koneksi Antar Komponen

Lakukan pengujian dasar untuk memastikan koneksi antar sistem berfungsi.

**1. Uji HDFS:**

```bash
hdfs dfs -ls /
```

**2. Uji Hive Metastore:**

```bash
hive --service metastore &
beeline -u jdbc:hive2://master:10000 -n hive_user -p hive_password -e "SHOW DATABASES;"
```

**3. Uji Spark:**

```bash
spark-submit --master yarn --deploy-mode client --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples_2.13-4.0.1.jar 10
```

**4. Uji ADLS Token:**

```bash
curl -X POST -d "grant_type=client_credentials&client_id=$ADLS_CLIENT_ID&client_secret=$ADLS_CLIENT_SECRET&resource=https://storage.azure.com/" https://login.microsoftonline.com/$ADLS_TENANT_ID/oauth2/token
```

Jika token diterbitkan dan status HTTP 200, autentikasi ADLS berhasil.

---

#### 8. Validasi di Airflow UI

Masuk kembali ke dashboard Airflow → **Admin → Connections**
Klik tombol **Test** di tiap koneksi (`hdfs_conn`, `spark_conn`, `hive_metastore_conn`, `adls_conn`).

Jika semua koneksi menampilkan status **Connection Test Succeeded**, konfigurasi dinyatakan valid.

---

#### 9. Hasil Akhir Sprint 4 Bagian 2

| Komponen              | Status   | Keterangan                                          |
| --------------------- | -------- | --------------------------------------------------- |
| Airflow Connections   | Aktif    | 4 koneksi utama (HDFS, Spark, Hive, ADLS) tersimpan |
| Airflow Variables     | Diset    | Path dan konfigurasi project INSIGHTERA aktif       |
| File `.env`           | Dibuat   | Menyimpan credential rahasia ADLS & Hive Metastore  |
| Environment Variables | Termuat  | Otomatis saat service Airflow dijalankan            |
| Pengujian Koneksi     | Sukses   | Semua hook antar sistem berfungsi normal            |
| Dashboard Airflow     | Akses OK | Semua koneksi dapat dites dari UI                   |

---

#### 10. Kesimpulan

Tahapan ini berhasil menyiapkan seluruh konfigurasi dasar yang diperlukan agar **Apache Airflow 3.1.0** dapat terhubung ke sistem hybrid INSIGHTERA (HDFS–Spark–Hive–ADLS).
Dengan koneksi dan variabel global ini, seluruh pipeline ETL yang akan dibuat pada Sprint 4 Bagian 3 dapat langsung memanfaatkan konfigurasi yang konsisten dan aman, tanpa menduplikasi credential atau path.

Langkah selanjutnya akan dilanjutkan pada **Sprint 4 – Bagian 3: Pembuatan DAG ETL Hybrid (Bronze → Silver → Gold)** untuk membangun pipeline otomatis yang mengorkestrasi alur data lakehouse INSIGHTERA.
