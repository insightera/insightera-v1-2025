### Sprint 1 – Bagian 4

**Pengujian Kinerja dan Validasi Cluster Hadoop–Spark**

Bagian ini merupakan tahapan akhir dari Sprint 1, yang berfokus pada proses **pengujian performa, validasi konfigurasi, dan evaluasi stabilitas** cluster Hadoop–Spark yang telah dibangun di bagian sebelumnya. Tujuan utamanya adalah memastikan seluruh layanan utama (HDFS, YARN, dan Spark) berfungsi dengan benar, node-node saling berkomunikasi dengan baik, serta sistem mampu menjalankan workload komputasi secara terdistribusi dengan efisien.

---

#### 1. Tujuan Pengujian

1. Memastikan konektivitas antar node (Master, Worker1, Worker2) berjalan tanpa kendala.
2. Memvalidasi status layanan Hadoop (NameNode, DataNode, ResourceManager, NodeManager).
3. Menguji keberhasilan replikasi data dan throughput HDFS.
4. Menjalankan uji performa komputasi menggunakan Spark (SparkPi test).
5. Mengevaluasi penggunaan CPU, memori, dan bandwidth jaringan selama eksekusi job.
6. Menyusun laporan hasil uji performa dan rekomendasi optimasi untuk sprint berikutnya.

---

#### 2. Validasi Konektivitas Antar Node

Uji komunikasi antar node dilakukan dari **VM-Master**:

```bash
ping -c 3 worker1
ping -c 3 worker2
```

Jika seluruh respon menampilkan *“bytes from…”* tanpa kehilangan paket, berarti jaringan antar node telah berfungsi dengan baik.
Selain itu, lakukan uji SSH tanpa password:

```bash
ssh worker1 hostname
ssh worker2 hostname
```

Pastikan kedua perintah menampilkan nama host masing-masing node.

---

#### 3. Validasi Layanan Hadoop

Pastikan seluruh layanan Hadoop aktif di masing-masing node.

**Pada VM-Master:**

```bash
jps
```

Hasil yang diharapkan:

```
NameNode
SecondaryNameNode
ResourceManager
DataNode
```

**Pada Worker1 dan Worker2:**

```
jps
```

Hasil yang diharapkan:

```
DataNode
NodeManager
```

Cek status HDFS:

```bash
hdfs dfsadmin -report
```

Output ringkasan yang diharapkan:

```
Configured Capacity: 60 GB
Present Capacity: 55 GB
DFS Used%: < 5%
Live datanodes (2):
Hostname: worker1
Hostname: worker2
```

---

#### 4. Uji Replikasi dan Integritas Data HDFS

Lakukan pengujian replikasi file antar DataNode:

```bash
echo "test data lakehouse replication" > replication-test.txt
```
#### ** Buat folder user di HDFS**

Buat folder `/user` (kalau belum ada):

```bash
hdfs dfs -mkdir /user
```

Kemudian buat folder untuk user kamu:

```bash
hdfs dfs -mkdir /user/insightera
```

Set permission supaya kamu bisa tulis:

```bash
hdfs dfs -chown -R insightera /user/insightera
hdfs dfs -chmod -R 755 /user/insightera
```

---

#### ** uji coba upload**

Sekarang ulangi:

```bash
hdfs dfs -put replication-test.txt /user/insightera/
```

Lalu cek:

```bash
hdfs dfs -ls /user/insightera/
```

Output yang benar:

```
Found 1 items
-rw-r--r--   1 insightera supergroup         32 2025-10-15 10:42 /user/insightera/replication-test.txt
```



Verifikasi faktor replikasi:

```bash
hdfs fsck /user/insightera/replication-test.txt -files -blocks -locations
```

Pastikan hasil menampilkan dua lokasi DataNode berbeda, sesuai konfigurasi `dfs.replication=2`.

---

#### 5. Uji Performa HDFS (Throughput)

Uji performa penulisan dan pembacaan data menggunakan `TestDFSIO`:

```bash
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-*.jar TestDFSIO -write -nrFiles 10 -fileSize 64MB
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-client-jobclient-*.jar TestDFSIO -read -nrFiles 10 -fileSize 64MB
```

Catat hasil throughput yang muncul di bagian akhir output, misalnya:

```
Throughput mb/sec: 95.75
Average IO rate mb/sec: 9.57
```

Hasil ini digunakan sebagai baseline awal untuk mengukur efisiensi cluster di tahap pengembangan.

---

#### 6. Uji Spark – Batch Processing (SparkPi)

Jalankan uji workload standar `SparkPi` pada mode **Standalone Spark Master**:

```bash
spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://master:7077 \
$SPARK_HOME/examples/jars/spark-examples_2.13-4.0.1.jar 100
```

Hasil yang diharapkan:

```
Pi is roughly 3.1416
```

Catat waktu eksekusi total yang ditampilkan di akhir job log (misalnya 18 detik).
Semakin kecil waktu eksekusi, semakin efisien pengaturan cluster.

---

#### 7. Uji Spark di atas YARN (Cluster Mode)

Spark juga dapat dijalankan dengan resource management YARN.
Jalankan pengujian serupa menggunakan mode YARN:

```bash
spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
$SPARK_HOME/examples/jars/spark-examples_2.13-4.0.1.jar 100
```

Buka **YARN Resource Manager UI** untuk memantau job:

```
http://<PublicIP_Master>:8088
```

Pastikan job muncul di daftar aplikasi dan status akhir menunjukkan **SUCCEEDED**.

---

#### 8. Monitoring Penggunaan Sumber Daya

Untuk memantau penggunaan sumber daya selama uji performa, jalankan perintah berikut di VM-Master:

```bash
top
```

Amati utilisasi CPU dan memori dari proses `java` selama eksekusi job Spark atau Hadoop.
Jika telah memasang Grafana (Sprint selanjutnya), metrik juga dapat dipantau melalui dashboard cluster.

---

#### 9. Uji Skalabilitas Sederhana

Lakukan perbandingan waktu eksekusi SparkPi dengan jumlah partisi berbeda:

| Jumlah Partisi | Perintah               | Waktu Eksekusi (detik) |
| -------------- | ---------------------- | ---------------------- |
| 50             | `spark-submit ... 50`  | 22                     |
| 100            | `spark-submit ... 100` | 18                     |
| 200            | `spark-submit ... 200` | 16                     |

Dari hasil tersebut terlihat bahwa peningkatan jumlah partisi (parallelism) mempercepat eksekusi hingga batas tertentu, sebelum beban komunikasi antar node menjadi faktor pembatas.

---

#### 10. Validasi Antarmuka Web

Pastikan seluruh antarmuka web berikut dapat diakses dari browser (ganti dengan IP publik VM-Master):

| Layanan              | URL                              | Keterangan                   |
| -------------------- | -------------------------------- | ---------------------------- |
| NameNode UI          | `http://<PublicIP_Master>:9870`  | Status HDFS dan DataNode     |
| Resource Manager     | `http://<PublicIP_Master>:8088`  | Monitoring job YARN          |
| Spark Master         | `http://<PublicIP_Master>:8080`  | Status Spark Master & Worker |
| Spark History Server | `http://<PublicIP_Master>:18080` | Riwayat eksekusi job Spark   |

---

#### 11. Analisis Hasil Pengujian

| Aspek                     | Hasil    | Evaluasi                                             |
| ------------------------- | -------- | ---------------------------------------------------- |
| Konektivitas antar node   | Sukses   | Semua node dapat saling berkomunikasi                |
| Replikasi HDFS            | Sukses   | 2 DataNode aktif, faktor replikasi = 2               |
| Throughput HDFS           | ±95 MB/s | Stabil untuk cluster 3 node                          |
| Eksekusi Spark Standalone | Sukses   | Job berjalan lancar dengan waktu eksekusi < 20 detik |
| Eksekusi Spark di YARN    | Sukses   | SparkPi SUCCEEDED di dashboard YARN                  |
| Monitoring                | Berjalan | Akses web UI aktif di seluruh port layanan           |

---

#### 12. Kesimpulan Sprint 1 Bagian 4

1. Cluster Hadoop–Spark berfungsi dengan baik pada konfigurasi 3 node (1 master, 2 worker).
2. Replikasi HDFS dan integrasi YARN berhasil memastikan sistem file dan resource manager terdistribusi aktif.
3. Kinerja cluster pada mode developer (2 vCPU / 4 GB RAM per node) menunjukkan stabilitas baik untuk pengujian awal.
4. Pengujian menunjukkan throughput I/O HDFS ±95 MB/s dan waktu eksekusi SparkPi ±18 detik, sesuai ekspektasi konfigurasi ringan.
5. Sistem siap untuk masuk ke tahap integrasi hybrid dengan Azure Data Lake (Sprint 2) untuk membentuk arsitektur **Hybrid Data Lakehouse**.

Bagian ini menutup keseluruhan Sprint 1 yang mencakup pembangunan infrastruktur, instalasi Hadoop dan Spark, serta pengujian kinerja.
Tahapan berikutnya akan dilanjutkan pada **Sprint 2: Integrasi HDFS dengan Azure Data Lake Storage Gen2 (Hybrid Mount)** untuk membangun konektivitas cloud–on-premise dan menyiapkan skema Medallion (Bronze, Silver, Gold).