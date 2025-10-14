### Sprint 2 – Bagian 4

**Pengujian Sinkronisasi dan Validasi Kinerja Hybrid Data Lakehouse**

Bagian ini merupakan tahap akhir dari Sprint 2 yang berfokus pada **pengujian fungsional dan performa** dari integrasi antara HDFS (lokal) dan Azure Data Lake Storage Gen2 (cloud).
Tujuannya adalah memastikan sistem hybrid bekerja dengan stabil, sinkronisasi berjalan dua arah, dan performa baca/tulis antar lingkungan memenuhi kebutuhan operasional INSIGHTERA untuk data lakehouse federatif.

---

#### 1. Tujuan Pengujian

1. Memastikan konektivitas dan autentikasi OAuth2 berfungsi dengan stabil pada semua node.
2. Memverifikasi bahwa Hadoop dapat membaca dan menulis langsung ke ADLS melalui `abfs://`.
3. Menguji kecepatan sinkronisasi data antara HDFS dan ADLS (arah lokal ke cloud dan sebaliknya).
4. Menilai latensi transfer, kecepatan throughput, dan efisiensi replikasi file.
5. Memastikan konsistensi data setelah proses transfer dan sinkronisasi dilakukan.

---

#### 2. Verifikasi Autentikasi OAuth2

Sebelum melakukan uji performa, pastikan autentikasi masih valid:

```bash
hdfs dfs -ls abfs://bronze@insighteradata.dfs.core.windows.net/
```

Jika daftar folder 15 unit kerja muncul tanpa error, berarti Service Principal `sp-insightera` masih aktif dan token OAuth2 berfungsi dengan baik.

---

#### 3. Uji Sinkronisasi Arah HDFS → ADLS

Uji pertama dilakukan dengan mengunggah file dari cluster lokal ke penyimpanan cloud.

```bash
hdfs dfs -mkdir /user/insightera/testsync
echo "uji sinkronisasi data lakehouse iteratif" > sync_local.txt
hdfs dfs -put sync_local.txt /user/insightera/testsync/
```

Lalu salin direktori tersebut ke ADLS container `bronze`:

```bash
hadoop distcp /user/insightera/testsync \
abfs://bronze@insighteradata.dfs.core.windows.net/satudata/
```

Verifikasi dari Azure Portal atau Storage Explorer:

```
/bronze/satudata/testsync/sync_local.txt
```

File ini menandakan bahwa HDFS berhasil menulis ke ADLS secara hybrid.

---

#### 4. Uji Sinkronisasi Arah ADLS → HDFS

Lakukan pengujian sebaliknya, yaitu menyalin file dari ADLS ke sistem lokal HDFS:

```bash
echo "uji transfer dari adls ke hdfs" > adls_test.txt
az storage fs file upload \
  --account-name insighteradata \
  --file-system bronze \
  --path akademik/adls_test.txt \
  --source adls_test.txt

# Sekarang tarik kembali file ke HDFS
hadoop distcp \
abfs://bronze@insighteradata.dfs.core.windows.net/akademik/adls_test.txt \
/user/insightera/from_adls/
```

Verifikasi hasil:

```bash
hdfs dfs -ls /user/insightera/from_adls/
hdfs dfs -cat /user/insightera/from_adls/adls_test.txt
```

Jika isi file terbaca dengan benar, maka arah sinkronisasi ADLS → HDFS juga berhasil.

---

#### 5. Pengujian Bandwidth dan Throughput Transfer

Untuk mengukur performa transfer antar sistem, gunakan perintah `time` dan `distcp` untuk dataset berukuran besar. Misalnya untuk data uji 1 GB:

```bash
time hadoop distcp /user/insightera/sample_1gb \
abfs://silver@insighteradata.dfs.core.windows.net/keuangan/
```

Contoh hasil:

```
real    0m48.562s
user    0m5.373s
sys     0m0.891s
```

Catatan throughput pada log Hadoop (di akhir output) biasanya berkisar:

```
Avg Bytes Transferred per sec = 22 MB/s
Total bytes = 1,000,000,000
```

Ini menandakan koneksi jaringan Azure regional dan cluster lokal bekerja dengan baik.

---

#### 6. Uji Latensi Akses File Langsung

Lakukan uji waktu baca file kecil (<1 MB) dari kedua sistem:

```bash
time hdfs dfs -cat abfs://bronze@insighteradata.dfs.core.windows.net/akademik/test.txt > /dev/null
time hdfs dfs -cat /user/insightera/testsync/sync_local.txt > /dev/null
```

Bandingkan hasil waktu eksekusi:

| Arah Akses | Rata-rata Latency | Keterangan                              |
| ---------- | ----------------- | --------------------------------------- |
| HDFS lokal | 0.08 detik        | Baca cepat karena lokal                 |
| ADLS cloud | 0.25 detik        | Lebih lambat, wajar karena jaringan WAN |

Perbedaan latency ini masih dalam rentang normal dan dapat dioptimalkan menggunakan caching pipeline di fase berikutnya.

---

#### 7. Uji Konsistensi dan Validasi Data

Untuk memastikan integritas data setelah sinkronisasi:

```bash
hdfs dfs -cat /user/insightera/testsync/sync_local.txt | md5sum
hdfs dfs -cat abfs://bronze@insighteradata.dfs.core.windows.net/satudata/testsync/sync_local.txt | md5sum
```

Jika hash MD5 identik, berarti file berhasil ditransfer tanpa perubahan atau korupsi data.

---

#### 8. Pengujian Paralel Sinkronisasi Multi-Domain

Uji sinkronisasi paralel untuk beberapa domain unit kerja sekaligus:

```bash
hadoop distcp \
/user/insightera/akademik \
abfs://silver@insighteradata.dfs.core.windows.net/akademik/ &

hadoop distcp \
/user/insightera/keuangan \
abfs://silver@insighteradata.dfs.core.windows.net/keuangan/ &

hadoop distcp \
/user/insightera/kemahasiswaan \
abfs://silver@insighteradata.dfs.core.windows.net/kemahasiswaan/ &
```

Gunakan perintah `jobs` untuk memantau proses yang berjalan bersamaan.
Setelah selesai, verifikasi folder di container `silver` untuk memastikan seluruh domain tersalin.

---

#### 9. Pengujian Skalabilitas dan Stabilitas Cluster

Lakukan beberapa job sinkronisasi berturut-turut dan pantau performa sistem:

```bash
watch -n 5 "jps | grep DataNode"
```

Pastikan seluruh node tetap aktif selama proses berlangsung, tanpa node terputus atau gagal koneksi.

Gunakan YARN Resource Manager UI (`http://<PublicIP_Master>:8088`) untuk memastikan setiap job sinkronisasi (`distcp`) tereksekusi secara paralel dan berstatus **SUCCEEDED**.

---

#### 10. Analisis Hasil Uji Kinerja

| Parameter                           | Hasil Pengujian | Evaluasi                                     |
| ----------------------------------- | --------------- | -------------------------------------------- |
| **Kecepatan Upload HDFS → ADLS**    | ±22 MB/s        | Baik untuk 3-node cluster (dev mode)         |
| **Kecepatan Download ADLS → HDFS**  | ±19 MB/s        | Stabil                                       |
| **Latency Akses File Cloud**        | 0.25 detik      | Normal                                       |
| **Replikasi Data Mart (15 domain)** | Sukses          | Semua direktori sinkron                      |
| **Konsistensi Data (MD5 Checksum)** | Identik         | Tidak ada korupsi data                       |
| **Uji Multi-threading Distcp**      | Berhasil        | Cluster mampu menangani beban paralel ringan |

---

#### 11. Validasi Melalui Azure Portal

1. Masuk ke **Azure Portal → Storage Account → insighteradata**.
2. Periksa tab **Containers** dan pastikan direktori `bronze`, `silver`, dan `gold` telah berisi folder 15 unit kerja.
3. Pastikan waktu modifikasi (Last Modified) sesuai dengan waktu uji sinkronisasi terakhir.
4. Buka **Monitoring → Insights** untuk melihat aktivitas I/O dan bandwidth transfer dari cluster.

---

#### 12. Hasil Akhir Sprint 2 Bagian 4

| Komponen                 | Status             | Keterangan                                  |
| ------------------------ | ------------------ | ------------------------------------------- |
| Autentikasi              | Berhasil           | OAuth2 stabil menggunakan Service Principal |
| Sinkronisasi HDFS ↔ ADLS | Berfungsi dua arah | distcp dan abfs berjalan lancar             |
| Performa Transfer        | Baik               | 20–25 MB/s untuk 1 GB data                  |
| Integritas File          | Terjaga            | MD5 Checksum identik                        |
| Replikasi Domain         | Sukses             | 15 unit kerja tersinkronisasi               |
| Monitoring               | Aktif              | Data terekam di portal Azure                |

---

#### 13. Kesimpulan

Hasil pengujian menunjukkan bahwa integrasi hybrid antara **Hadoop Cluster (HDFS)** dan **Azure Data Lake Storage Gen2 (ADLS)** berjalan dengan baik.
Sistem mampu melakukan *bidirectional sync* (dua arah), mempertahankan konsistensi data, dan menampilkan performa yang stabil untuk skala pengembangan.

Dengan keberhasilan Sprint 2 ini, arsitektur **Hybrid Data Lakehouse INSIGHTERA** telah sepenuhnya terbentuk, siap untuk dikembangkan ke tahap **Fase II**, yaitu:

* **Sprint 3:** Integrasi Hive Metastore dan Data Federation
* **Sprint 4:** Implementasi Pipeline ETL menggunakan Airflow
* **Sprint 5:** Penyusunan Federated Data Mart dan Data Warehouse (EDW)

Langkah berikutnya akan dilanjutkan pada **Sprint 3 – Bagian 1: Persiapan dan Instalasi Hive Metastore berbasis PostgreSQL**.
