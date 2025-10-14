### Sprint 1 – Bagian 3b

**Instalasi dan Konfigurasi Apache Spark 4.0.1 pada VM Master, Worker1, dan Worker2**

Tahapan ini melanjutkan konfigurasi Hadoop yang telah selesai di bagian sebelumnya. Tujuannya adalah memasang dan mengonfigurasi **Apache Spark versi 4.0.1** pada seluruh node (Master dan Worker) agar dapat berintegrasi langsung dengan HDFS serta menjalankan pekerjaan komputasi terdistribusi berbasis YARN. Spark akan menjadi komponen utama lapisan komputasi INSIGHTERA yang digunakan untuk analitik data, transformasi batch, dan integrasi dengan pipeline Airflow.

---

#### 1. Lingkup Instalasi

| Node           | Peran                                    | Komponen Spark                          |
| -------------- | ---------------------------------------- | --------------------------------------- |
| **VM-Master**  | Spark Master, Driver, dan History Server | Menjalankan koordinasi dan eksekusi job |
| **VM-Worker1** | Spark Worker                             | Menjalankan eksekusi task pada cluster  |
| **VM-Worker2** | Spark Worker                             | Menjalankan eksekusi task pada cluster  |

---

#### 2. Persiapan Lingkungan

Pastikan Hadoop sudah berjalan normal pada semua node. Jalankan perintah berikut untuk memastikan status HDFS aktif:

```bash
hdfs dfsadmin -report
```

Pastikan ada dua DataNode aktif. Setelah itu, lakukan pembaruan sistem dan instalasi dependensi tambahan di semua node:

```bash
sudo apt update -y
sudo apt install openjdk-11-jdk scala git -y
```

Verifikasi Java dan Scala:

```bash
java -version
scala -version
```

---

#### 3. Unduh dan Ekstrak Apache Spark

Lakukan langkah berikut di setiap node (Master dan dua Worker):

```bash
cd /opt
sudo wget https://downloads.apache.org/spark/spark-4.0.1/spark-4.0.1-bin-hadoop3.tgz
sudo tar -xzf spark-4.0.1-bin-hadoop3.tgz
sudo mv spark-4.0.1-bin-hadoop3 spark
sudo chown -R insightera:insightera /opt/spark
```

Tambahkan variabel lingkungan Spark pada file `~/.bashrc`:

```bash
echo "export SPARK_HOME=/opt/spark" >> ~/.bashrc
echo "export PATH=\$PATH:\$SPARK_HOME/bin:\$SPARK_HOME/sbin" >> ~/.bashrc
echo "export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop" >> ~/.bashrc
echo "export SPARK_DIST_CLASSPATH=\$(hadoop classpath)" >> ~/.bashrc
source ~/.bashrc
```

Verifikasi instalasi Spark:

```bash
spark-shell --version
```

---

#### 4. Konfigurasi Spark Environment

Masuk ke direktori konfigurasi Spark:

```bash
cd /opt/spark/conf
```

Salin template file konfigurasi:

```bash
cp spark-env.sh.template spark-env.sh
cp workers.template workers
```

Edit file `spark-env.sh`:

```bash
nano spark-env.sh
```

Isi dengan konfigurasi berikut:

```bash
export SPARK_MASTER_HOST=master
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HADOOP_CONF_DIR=/opt/hadoop/etc/hadoop
export SPARK_WORKER_CORES=2
export SPARK_WORKER_MEMORY=2g
export SPARK_DRIVER_MEMORY=1g
export SPARK_EXECUTOR_MEMORY=2g
export SPARK_HISTORY_OPTS="-Dspark.history.fs.logDirectory=hdfs://master:9000/spark-logs"
```

Kemudian edit file `workers` dan isi dengan:

```
worker1
worker2
```

Salin file konfigurasi ini dari Master ke kedua Worker:

```bash
scp -r /opt/spark/conf worker1:/opt/spark/
scp -r /opt/spark/conf worker2:/opt/spark/
```

---

#### 5. Menyiapkan Direktori Log Spark di HDFS

Buat direktori log Spark di HDFS agar History Server dapat menyimpan hasil eksekusi job:

```bash
hdfs dfs -mkdir /spark-logs
hdfs dfs -chmod -R 777 /spark-logs
```

---

#### 6. Menjalankan Spark Cluster

Langkah ini dilakukan di VM-Master:

```bash
start-master.sh
start-workers.sh
```

Untuk memastikan semua komponen berjalan, jalankan perintah berikut di semua node:

```bash
jps
```

Hasil yang diharapkan:

* **Master Node:** Master, Worker
* **Worker1 & Worker2:** Worker

Akses antarmuka web Spark:

* Spark Master UI: `http://<PublicIP_Master>:8080`

---

#### 7. Menjalankan History Server

Di VM-Master, aktifkan History Server untuk memantau eksekusi job Spark:

```bash
start-history-server.sh
```

Akses melalui browser:

```
http://<PublicIP_Master>:18080
```

---

#### 8. Pengujian Spark Cluster

Lakukan uji job sederhana (SparkPi):

```bash
spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://master:7077 \
$SPARK_HOME/examples/jars/spark-examples_2.13-4.0.1.jar 100
```

Jika hasil output menampilkan nilai Pi seperti:

```
Pi is roughly 3.1418
```

maka Spark telah berjalan dengan benar di seluruh cluster.

---

#### 9. Integrasi Spark dengan YARN

Spark juga dapat dijalankan di atas Hadoop YARN. Untuk pengujian, jalankan:

```bash
spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
$SPARK_HOME/examples/jars/spark-examples_2.13-4.0.1.jar 100
```

Verifikasi hasil job melalui antarmuka YARN:

```
http://<PublicIP_Master>:8088
```

Pastikan aplikasi Spark muncul di daftar job yang sedang atau telah dijalankan.

---

#### 10. Validasi Akhir

| Komponen             | Node       | Status   | Keterangan                      |
| -------------------- | ---------- | -------- | ------------------------------- |
| Spark Master         | master     | Aktif    | Dapat menerima koneksi worker   |
| Spark Worker 1       | worker1    | Aktif    | Terdaftar pada Spark Master     |
| Spark Worker 2       | worker2    | Aktif    | Terdaftar pada Spark Master     |
| Spark History Server | master     | Aktif    | Mencatat log job dari HDFS      |
| Integrasi HDFS       | Semua node | Berhasil | Spark membaca data dari HDFS    |
| Integrasi YARN       | Semua node | Berhasil | Spark job berjalan di atas YARN |

---

#### 11. Catatan Konfigurasi dan Optimasi

1. Untuk cluster kecil (developer mode), gunakan parameter `SPARK_WORKER_MEMORY=2g` agar stabil.
2. Saat menggunakan YARN, pastikan `yarn.nodemanager.resource.memory-mb` disesuaikan dengan memori VM (mis. 4096 MB).
3. Aktifkan fitur **auto-shutdown** di Azure agar biaya VM tetap efisien selama fase pengujian.
4. Backup konfigurasi Spark dan Hadoop di direktori:

   ```
   /opt/insightera/config/
   ```

   agar mudah digunakan untuk replikasi cluster berikutnya.

---

Sprint 1 Bagian 3b ini menandai selesainya pembangunan lapisan komputasi hybrid INSIGHTERA berbasis Hadoop dan Spark. Seluruh node kini siap digunakan untuk pengujian pipeline data lakehouse dan integrasi dengan Azure Data Lake pada Sprint 2 berikutnya.
Langkah berikutnya akan dilanjutkan pada **Sprint 2: Integrasi HDFS dengan Azure Data Lake Storage (Hybrid Mount)**.
