## **A. Port Utama Hadoop (Core Cluster)**

| Komponen                            | Port      | Akses di Browser         | Keterangan                                          |
| ----------------------------------- | --------- | ------------------------ | --------------------------------------------------- |
| **HDFS NameNode UI**                | **9870**  | `http://localhost:9870`  | Melihat status HDFS, kapasitas, DataNode, file tree |
| **HDFS DataNode UI**                | **9864**  | `http://localhost:9864`  | Monitoring DataNode (aktif di setiap worker)        |
| **Secondary NameNode UI**           | **9868**  | `http://localhost:9868`  | Monitoring checkpoint NameNode                      |
| **YARN ResourceManager UI**         | **8088**  | `http://localhost:8088`  | Monitoring job, queue, aplikasi Spark & MapReduce   |
| **YARN NodeManager UI**             | **8042**  | `http://localhost:8042`  | Monitoring container & task di tiap node            |
| **JobHistory Server (MapReduce)**   | **19888** | `http://localhost:19888` | Melihat riwayat eksekusi job MapReduce              |
| **Spark Web UI (driver)**           | **4040**  | `http://localhost:4040`  | Menampilkan Spark jobs, stages, dan executor        |
| **Spark History Server (opsional)** | **18080** | `http://localhost:18080` | Melihat log & hasil Spark jobs terdahulu            |

---

## **B. Port Pendukung (Opsional / Service Tambahan)**

| Komponen                       | Port      | Keterangan                                |
| ------------------------------ | --------- | ----------------------------------------- |
| **HiveServer2 (JDBC/Beeline)** | **10000** | Untuk koneksi SQL ke Hive                 |
| **Hive Metastore**             | **9083**  | Dihubungkan dengan PostgreSQL metastore   |
| **Airflow Webserver**          | **8080**  | UI workflow DAG (ada di VM-Management)    |
| **Grafana**                    | **3000**  | Dashboard visualisasi cluster & metrik    |
| **PostgreSQL**                 | **5432**  | Database metastore Hive / Airflow backend |
| **Prometheus (opsional)**      | **9090**  | Monitoring metrics untuk Hadoop/Spark     |

---

## **C. Port Forwarding dari Laptop ke VM Management → VM Master**

Jika kamu ingin mengakses semuanya **dari laptop lokal (browser)**, cukup jalankan tunnel seperti ini dari laptop 👇

```bash
ssh -i ~/.ssh/insightera \
  -L 9870:master:9870 \
  -L 8088:master:8088 \
  -L 4040:master:4040 \
  -L 18080:master:18080 \
  -L 19888:master:19888 \
  -L 9864:worker1:9864 \
  -L 9864:worker2:9864 \
  insightera@20.184.7.134
```

Lalu di browser lokal buka:

* 🌐 `http://localhost:9870` → NameNode
* 🌐 `http://localhost:8088` → ResourceManager
* 🌐 `http://localhost:4040` → Spark session aktif
* 🌐 `http://localhost:9864` → DataNode
* 🌐 `http://localhost:18080` → Spark History

---

## Tips Tambahan

1. **Pastikan di VM-Master firewall/NSG tidak memblokir port internal** (9864, 9870, 8088, dsb).
   Jalankan:

   ```bash
   sudo ufw disable
   ```

   (karena NSG Azure sudah handle keamanan antar-VM).

2. **Bisa buat alias tunnel di `~/.ssh/config`**:

   ```bash
   Host tunnel-hadoop
       HostName 20.184.7.134
       User insightera
       IdentityFile ~/.ssh/insightera
       LocalForward 9870 master:9870
       LocalForward 8088 master:8088
       LocalForward 4040 master:4040
       LocalForward 18080 master:18080
       LocalForward 19888 master:19888
       LocalForward 9864 worker1:9864
       LocalForward 9865 worker2:9864
   ```
