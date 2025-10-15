### Sprint 4 – Bagian 3

**Pembuatan DAG ETL Hybrid (Bronze → Silver → Gold) untuk Pipeline INSIGHTERA**

Bagian ini merupakan inti dari **Sprint 4**, di mana sistem **Apache Airflow 3.1.0** yang telah dikonfigurasi kini digunakan untuk membangun pipeline ETL otomatis yang mengalirkan data antar lapisan **Bronze → Silver → Gold** dalam arsitektur *Hybrid Data Lakehouse* INSIGHTERA.

Pipeline ini mengorkestrasi tiga tahap utama:

1. **Ingestion** – mengambil data mentah dari sumber internal/eksternal ke layer Bronze.
2. **Transformation** – membersihkan dan menstandarkan data dari Bronze ke Silver.
3. **Aggregation** – menggabungkan data hasil transformasi menjadi dataset Gold yang siap untuk analisis dan visualisasi.

Semua proses dijalankan otomatis menggunakan **Airflow DAG (Directed Acyclic Graph)** dan task-task terhubung antar sistem Hadoop, Spark, Hive, dan ADLS.

---

#### 1. Tujuan Bagian Ini

1. Membuat tiga DAG utama untuk setiap tahap pipeline (Bronze–Silver–Gold).
2. Mengintegrasikan operator `BashOperator`, `SparkSubmitOperator`, dan `HiveOperator`.
3. Mengimplementasikan dependensi antar task (`ingest → transform → aggregate`).
4. Menjadwalkan pipeline agar berjalan otomatis (harian / batch).
5. Menguji eksekusi dan validasi hasil pipeline di ADLS.

---

#### 2. Struktur Pipeline ETL INSIGHTERA

Setiap pipeline mengikuti struktur tetap sebagai berikut:

```
DAG_ETL_INSIGHTERA
│
├── Task 1: Ingest Raw Data (Bronze)
│     ├── Menyalin file dari sumber eksternal/API/DB lokal ke ADLS bronze
│     └── Menjalankan validasi struktur awal
│
├── Task 2: Transform Clean Data (Silver)
│     ├── Spark job untuk membersihkan dan menormalisasi data
│     └── Hasil disimpan di ADLS silver
│
└── Task 3: Aggregate Data (Gold)
      ├── Hive query untuk membuat data agregat
      └── Output: tabel Gold siap untuk visualisasi
```

---

#### 3. Pembuatan File DAG Airflow

Buat file:
`/opt/airflow/dags/etl_insightera_pipeline.py`

Isi dengan kode berikut:

```python
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator
from airflow.providers.apache.hive.operators.hive import HiveOperator
from airflow.models import Variable
from datetime import datetime

# Variabel global
BRONZE_PATH = Variable.get("BRONZE_PATH")
SILVER_PATH = Variable.get("SILVER_PATH")
GOLD_PATH = Variable.get("GOLD_PATH")
SPARK_HOME = Variable.get("SPARK_HOME")
PROJECT_NAME = Variable.get("PROJECT_NAME")

default_args = {
    'owner': 'insightera',
    'start_date': datetime(2025, 8, 26),
    'retries': 1,
}

with DAG(
    dag_id='etl_pipeline_insightera',
    default_args=default_args,
    schedule_interval='@daily',
    catchup=False,
    tags=['insightera', 'etl', 'hybrid']
) as dag:

    # 1. Ingestion Task
    ingest_bronze = BashOperator(
        task_id='ingest_raw_to_bronze',
        bash_command=f"""
        echo "Starting ingestion to Bronze Layer...";
        hdfs dfs -mkdir -p /user/insightera/bronze/input &&
        hdfs dfs -put -f /opt/data/source/*.csv /user/insightera/bronze/input/ &&
        echo "Data ingested to HDFS Bronze Layer."
        """
    )

    # 2. Transformation Task (Spark)
    transform_silver = SparkSubmitOperator(
        task_id='transform_bronze_to_silver',
        conn_id='spark_conn',
        application=f"{SPARK_HOME}/jobs/transform_bronze_to_silver.py",
        application_args=[
            f"--bronze_path={BRONZE_PATH}",
            f"--silver_path={SILVER_PATH}"
        ],
        executor_cores=2,
        executor_memory='2g',
        driver_memory='2g',
        verbose=True
    )

    # 3. Aggregation Task (Hive)
    aggregate_gold = HiveOperator(
        task_id='aggregate_silver_to_gold',
        hql=f"""
        CREATE TABLE IF NOT EXISTS gold.analytics_summary AS
        SELECT kategori, COUNT(*) AS jumlah_record
        FROM silver.cleaned_data
        GROUP BY kategori;
        """,
        hive_cli_conn_id='hive_metastore_conn'
    )

    # Task Dependencies
    ingest_bronze >> transform_silver >> aggregate_gold
```

---

#### 4. Membuat Script Transformasi PySpark

Buat file:
`/opt/spark/jobs/transform_bronze_to_silver.py`

Isi contoh script transformasi sederhana:

```python
from pyspark.sql import SparkSession
import argparse

parser = argparse.ArgumentParser()
parser.add_argument("--bronze_path", required=True)
parser.add_argument("--silver_path", required=True)
args = parser.parse_args()

spark = SparkSession.builder.appName("Transform Bronze to Silver").getOrCreate()

# Baca data mentah dari ADLS Bronze
df = spark.read.option("header", True).csv(f"{args.bronze_path}/raw/")

# Bersihkan data (contoh: hapus baris kosong)
df_clean = df.dropna()

# Simpan hasil ke ADLS Silver
df_clean.write.mode("overwrite").parquet(f"{args.silver_path}/cleaned/")

spark.stop()
print("Transformasi Bronze → Silver selesai.")
```

Script ini contoh sederhana untuk menampilkan format pipeline yang bisa dikembangkan lebih lanjut untuk tiap domain data mart di fase berikutnya.

---

#### 5. Menjalankan Pipeline

Aktifkan DAG di Airflow UI:

1. Masuk ke `http://<Public-IP-VM-Management>:8080`
2. Buka menu *DAGs → etl_pipeline_insightera*
3. Klik tombol **Trigger DAG**

Pantau jalannya pipeline melalui menu **Graph View**.
Setiap task (Ingest → Transform → Aggregate) akan dieksekusi berurutan.

Verifikasi hasil di ADLS:

* Folder `/bronze/input/` berisi file mentah.
* Folder `/silver/cleaned/` berisi file parquet hasil transformasi.
* Hive table `gold.analytics_summary` berisi data agregat.

---

#### 6. Pengujian Pipeline Manual (CLI)

Jalankan DAG secara manual dari terminal untuk pengujian cepat:

```bash
airflow dags trigger etl_pipeline_insightera
airflow tasks list etl_pipeline_insightera
airflow tasks test etl_pipeline_insightera transform_bronze_to_silver 2025-08-26
```

Cek log untuk memastikan semua task sukses:

```bash
airflow tasks logs etl_pipeline_insightera aggregate_silver_to_gold 2025-08-26
```

---

#### 7. Monitoring Eksekusi dan Logging

Airflow otomatis menyimpan log setiap task ke folder:

```
/opt/airflow/logs/etl_pipeline_insightera/
```

Dari Web UI, kamu dapat memantau:

* **Gantt Chart** – waktu eksekusi antar task
* **Task Logs** – output dan error detail dari Spark/Hive
* **Tree View** – status sukses atau gagal untuk tiap task

---

#### 8. Hasil Akhir Sprint 4 Bagian 3

| Komponen            | Status    | Keterangan                             |
| ------------------- | --------- | -------------------------------------- |
| DAG ETL INSIGHTERA  | Dibuat    | DAG utama untuk Bronze → Silver → Gold |
| SparkSubmitOperator | Aktif     | Menjalankan job PySpark transformasi   |
| HiveOperator        | Aktif     | Membuat tabel agregat di layer Gold    |
| BashOperator        | Aktif     | Menangani ingestion ke layer Bronze    |
| Pipeline            | Berfungsi | Ketiga tahap ETL berjalan otomatis     |
| UI Monitoring       | OK        | DAG tampil dan berjalan di Airflow Web |

---

#### 9. Kesimpulan

Tahapan ini berhasil membangun **pipeline ETL otomatis INSIGHTERA** yang mengalirkan data secara hybrid dari **HDFS (Bronze)** ke **ADLS (Silver dan Gold)**, dikendalikan oleh **Apache Airflow 3.1.0** dengan dukungan Spark dan Hive Operator.
Pipeline ini menjadi pondasi sistem *Data Lakehouse Orchestration* INSIGHTERA dan siap diperluas untuk domain-domain data mart pada Fase II.

Langkah berikutnya akan dilanjutkan pada **Sprint 4 – Bagian 4: Pengujian, Monitoring, dan Validasi Kinerja Pipeline ETL Hybrid.**
