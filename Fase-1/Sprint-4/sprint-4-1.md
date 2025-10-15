# Sprint 4 – Bagian 1

**Instalasi dan Konfigurasi Apache Airflow 3.1.0 pada VM Management (Hybrid ETL Orchestration)**

Tahapan ini berfokus pada pemasangan dan pengaturan **Apache Airflow 3.1.0** di VM Management, yang berfungsi sebagai pusat orkestrasi pipeline ETL INSIGHTERA.
Airflow akan mengatur proses *ingest*, *transform*, dan *aggregate* dari lapisan **Bronze → Silver → Gold** di ekosistem hybrid Hadoop–Spark–ADLS–Hive, serta menjadi *scheduler* dan *monitoring layer* untuk semua proses ETL yang dijalankan secara otomatis.

---

## 1. Tujuan Bagian Ini

1. Menginstal **Apache Airflow 3.1.0** dengan dukungan **Python 3.12**.
2. Mengonfigurasi Airflow menggunakan **PostgreSQL** sebagai metadata database backend.
3. Mengaktifkan mode eksekusi **LocalExecutor** untuk task paralel.
4. Menyiapkan direktori proyek Airflow (`dags/`, `logs/`, `plugins/`).
5. Menjalankan webserver, scheduler, dan memastikan UI Airflow dapat diakses.
6. Menghubungkan Airflow dengan Hadoop, Hive, Spark, dan ADLS.

---

## 2. Persiapan Lingkungan

Jalankan seluruh langkah di **VM Management** (yang sudah memiliki akses SSH ke cluster Hadoop dan ADLS).

Update sistem dan install dependency:

```bash
sudo apt update -y
sudo apt install python3.12 python3.12-venv python3.12-dev libpq-dev postgresql postgresql-contrib -y
```

Buat direktori Airflow dan environment virtual:

```bash
mkdir -p /opt/airflow
cd /opt/airflow
python3.12 -m venv venv
source venv/bin/activate
```

---

## 3. Instalasi Apache Airflow 3.1.0

Gunakan constraints file resmi agar seluruh dependency cocok dengan Python 3.12:

```bash
AIRFLOW_VERSION=3.1.0
PYTHON_VERSION=3.12
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

Verifikasi versi:

```bash
airflow version
```

Hasil:

```
3.1.0
```

---

## 4. Konfigurasi PostgreSQL sebagai Backend Metadata

Masuk ke shell PostgreSQL:

```bash
sudo -u postgres psql
```

Buat database dan user khusus Airflow:

```sql
CREATE DATABASE airflow_db;
CREATE USER airflow_user WITH PASSWORD 'airflow_password';
GRANT ALL PRIVILEGES ON DATABASE airflow_db TO airflow_user;
\q
```

---

## 5. Inisialisasi Airflow Home dan Database

Set environment variable:

```bash
export AIRFLOW_HOME=/opt/airflow
```

Inisialisasi konfigurasi default:

```bash
airflow db init
```

Kemudian ubah file konfigurasi utama:

```bash
nano $AIRFLOW_HOME/airflow.cfg
```

Cari dan ubah baris berikut:

```
executor = LocalExecutor
sql_alchemy_conn = postgresql+psycopg2://airflow_user:airflow_password@localhost:5432/airflow_db
```

Tambahkan pengaturan timezone dan parallelism agar sesuai kebutuhan ETL INSIGHTERA:

```
default_timezone = Asia/Jakarta
parallelism = 16
dag_concurrency = 8
max_active_runs_per_dag = 1
```

---

## 6. Membuat Akun Admin Airflow

Jalankan perintah berikut:

```bash
airflow users create \
  --username admin \
  --firstname Ardika \
  --lastname Satria \
  --role Admin \
  --email admin@insightera.itera.ac.id \
  --password admin123
```

---

## 7. Struktur Direktori Proyek Airflow

Pastikan struktur proyek lengkap:

```
/opt/airflow/
├── dags/
│   ├── etl_bronze_to_silver.py
│   ├── etl_silver_to_gold.py
│   └── validation_pipeline.py
├── logs/
├── plugins/
├── venv/
└── airflow.cfg
```

Direktori `dags/` akan digunakan untuk pipeline ETL hybrid pada Sprint 4 Bagian 3.

---

## 8. Menjalankan Layanan Airflow

Pastikan database sudah di-*migrate* ulang:

```bash
airflow db reset
airflow db init
```

Jalankan layanan utama:

```bash
airflow webserver --port 8080 -D
airflow scheduler -D
```

Verifikasi proses berjalan:

```bash
ps aux | grep airflow
```

Buka dashboard web:

```
http://<Public-IP-VM-Management>:8080
```

Login menggunakan akun `admin` yang telah dibuat.

---

## 9. Menambahkan Koneksi Hybrid ke Hadoop–Spark–Hive–ADLS

Masuk ke menu **Admin → Connections → + Add Connection** dan tambahkan berikut:

| Connection ID         | Type            | Host                                | Port | Extra JSON / URI                                                                              |
| --------------------- | --------------- | ----------------------------------- | ---- | --------------------------------------------------------------------------------------------- |
| `hdfs_conn`           | HDFS            | master                              | 9000 | `{ "user": "hadoop" }`                                                                        |
| `spark_conn`          | Spark           | master                              | 7077 | `{ "deploy-mode": "cluster" }`                                                                |
| `hive_metastore_conn` | Hive Metastore  | master                              | 9083 | `{ "schema": "hive_metastore" }`                                                              |
| `adls_conn`           | Azure Data Lake | insighteradata.dfs.core.windows.net | 443  | `{ "tenant": "<tenant_id>", "client_id": "<client_id>", "client_secret": "<client_secret>" }` |

Koneksi ini akan digunakan oleh operator Airflow (misalnya `SparkSubmitOperator`, `HiveOperator`, `WasbHook`, dll.) untuk mengakses sistem hybrid INSIGHTERA.

---

## 10. Pengujian DAG Dasar

Buat DAG sederhana di `/opt/airflow/dags/test_insightera_dag.py`:

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime

with DAG(
    dag_id='test_insightera_dag',
    schedule_interval='@daily',
    start_date=datetime(2025, 8, 25),
    catchup=False,
    tags=['insightera', 'test']
) as dag:

    hello = BashOperator(
        task_id='hello_task',
        bash_command='echo "Apache Airflow 3.1.0 for INSIGHTERA is running successfully!"'
    )

    hello
```

Buka Airflow UI → *DAGs → test_insightera_dag* → klik **Trigger DAG**, dan pastikan status berubah menjadi **Success**.

---

## 11. Hasil Akhir Sprint 4 Bagian 1

| Komponen              | Status        | Keterangan                                     |
| --------------------- | ------------- | ---------------------------------------------- |
| Apache Airflow        | Aktif         | Versi 3.1.0 dengan Python 3.12                 |
| Database Backend      | PostgreSQL    | `airflow_db` berjalan stabil                   |
| Executor              | LocalExecutor | Paralel 16 task                                |
| Webserver & Scheduler | Berjalan      | Port 8080 aktif                                |
| Integrasi Hybrid      | Siap          | Koneksi ke Hadoop, Spark, Hive, ADLS tersimpan |
| DAG Uji               | Sukses        | Task berjalan di UI Airflow                    |

---

## 12. Kesimpulan

Tahapan ini berhasil menginstal dan mengonfigurasi **Apache Airflow 3.1.0** sebagai sistem orkestrasi ETL utama di lingkungan INSIGHTERA dengan backend PostgreSQL dan integrasi penuh ke Hadoop–Spark–ADLS.
Airflow kini siap digunakan untuk **mengelola pipeline Bronze → Silver → Gold secara otomatis**, termasuk penjadwalan, logging, monitoring, dan eksekusi terdistribusi.

Langkah berikutnya akan dilanjutkan pada **Sprint 4 – Bagian 2: Konfigurasi Connection, Variable, dan Environment untuk Pipeline Hybrid INSIGHTERA.**
