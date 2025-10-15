### Sprint 1 – Bagian 3a

**Instalasi dan Konfigurasi Hadoop 3.4.2 pada VM Master, Worker1, dan Worker2**

Tahapan ini berfokus pada pemasangan dan konfigurasi Hadoop di seluruh node utama (Master dan dua Worker) untuk membangun sistem file terdistribusi (HDFS) dan resource manager (YARN). Hadoop akan menjadi fondasi utama lapisan komputasi INSIGHTERA yang menjalankan proses penyimpanan dan manajemen sumber daya secara terdistribusi.

---

#### 1. Lingkup Instalasi

Hadoop akan diinstal pada tiga node utama:

| Node           | Peran                                        | Layanan Hadoop                                     |
| -------------- | -------------------------------------------- | -------------------------------------------------- |
| **VM-Master**  | NameNode, SecondaryNameNode, ResourceManager | Mengelola metadata file dan koordinasi job         |
| **VM-Worker1** | DataNode, NodeManager                        | Menyimpan blok data dan menjalankan task komputasi |
| **VM-Worker2** | DataNode, NodeManager                        | Menyimpan blok data dan menjalankan task komputasi |

---

#### 2. Persiapan Lingkungan dan User

Masuk ke masing-masing node (Master, Worker1, Worker2) menggunakan SSH:

```bash
ssh -i ~/.ssh/insightera insightera@<PublicIP_VM>
```

Kemudian, pada setiap VM jalankan perintah berikut untuk menyiapkan direktori kerja Hadoop:

```bash
sudo apt update -y
sudo apt install openjdk-17-jdk -y
```

Verifikasi Java:

```bash
java -version
```

Output yang diharapkan:

```
openjdk version "17.0.xx"
```

Tambahkan variabel lingkungan Java ke `.bashrc`:

```bash
echo "export JAVA_HOME=$(readlink -f /usr/bin/java | sed 's:bin/java::')" >> ~/.bashrc
echo "export PATH=\$PATH:\$JAVA_HOME/bin" >> ~/.bashrc
source ~/.bashrc
```

---

#### 3. Unduh dan Ekstrak Hadoop

Lakukan langkah berikut pada setiap node:

```bash
cd /opt
sudo wget https://downloads.apache.org/hadoop/common/hadoop-3.4.2/hadoop-3.4.2.tar.gz
sudo tar -xzf hadoop-3.4.2.tar.gz
sudo mv hadoop-3.4.2 hadoop
sudo chown -R insightera:insightera /opt/hadoop
```

Tambahkan konfigurasi variabel lingkungan Hadoop di file `~/.bashrc`:

```bash
echo "export HADOOP_HOME=/opt/hadoop" >> ~/.bashrc
echo "export PATH=\$PATH:\$HADOOP_HOME/bin:\$HADOOP_HOME/sbin" >> ~/.bashrc
echo "export HADOOP_CONF_DIR=\$HADOOP_HOME/etc/hadoop" >> ~/.bashrc
source ~/.bashrc
```

Verifikasi instalasi:

```bash
hadoop version
```

---

#### 4. Konfigurasi SSH Antar Node

Agar Hadoop dapat beroperasi tanpa password antar node, lakukan pengaturan SSH dari VM-Master:

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id worker1
ssh-copy-id worker2
```

Lakukan uji koneksi:

```bash
ssh worker1
exit
ssh worker2
exit
```

Jika berhasil tanpa diminta password, konfigurasi SSH sudah benar.

---

#### 5. Konfigurasi File Hadoop

Seluruh konfigurasi berada di direktori `/opt/hadoop/etc/hadoop/`.
Edit file-file berikut di **VM-Master**, lalu salin ke Worker1 dan Worker2 menggunakan `scp`.

##### a. File `core-site.xml`

```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
  </property>
</configuration>
```

##### b. File `hdfs-site.xml`

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/opt/hadoop_data/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/opt/hadoop_data/hdfs/datanode</value>
  </property>
</configuration>
```

##### c. File `yarn-site.xml`

```xml
<configuration>
  <property>
    <name>yarn.resourcemanager.hostname</name>
    <value>master</value>
  </property>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```

##### d. File `mapred-site.xml`

Salin template:

```bash
cp /opt/hadoop/etc/hadoop/mapred-site.xml.template /opt/hadoop/etc/hadoop/mapred-site.xml
```

Kemudian ubah:

```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
</configuration>
```

##### e. File `workers`

Isi dengan:

```
worker1
worker2
```

Salin semua konfigurasi ke node Worker:

```bash
ssh worker1 "sudo mkdir -p /opt/hadoop/etc && sudo chown -R insightera:insightera /opt/hadoop"
ssh worker2 "sudo mkdir -p /opt/hadoop/etc && sudo chown -R insightera:insightera /opt/hadoop"
```

```bash
scp -r /opt/hadoop/etc/hadoop/ worker1:/opt/hadoop/etc/
scp -r /opt/hadoop/etc/hadoop/ worker2:/opt/hadoop/etc/
```
Edit /opt/hadoop/etc/hadoop/workers atau slaves

```bash
sudo nano /opt/hadoop/etc/hadoop/workers
```

lalu tambahkan:
```bash
worker1
worker2
```
### **Pastikan SSH Passwordless di Master**

Masuk ke VM-Master:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Uji:

```bash
ssh localhost "hostname"
```

Kalau sukses tanpa password → beres untuk lokal.
Lalu sebarkan ke semua worker:

```bash
for node in master worker1 worker2; do
  ssh-copy-id -i ~/.ssh/id_rsa.pub insightera@$node
done
```

Uji koneksi semua node:

```bash
for node in master worker1 worker2; do
  ssh $node "hostname"
done
```

---

### ** Pastikan Hadoop terpasang di lokasi yang sama di semua node**

Di setiap node (worker1, worker2), pastikan:

```bash
ls /opt/hadoop/bin/hdfs
```

Kalau hasilnya `No such file or directory`, berarti Hadoop belum dikopi ke situ.

💡 Solusi cepat (dari Master):

```bash
for node in worker1 worker2; do
  ssh $node "sudo mkdir -p /opt && sudo chown -R insightera:insightera /opt"
  scp -r /opt/hadoop $node:/opt/
done
```




Lalu verifikasi di setiap worker:

```bash
ssh worker1 "ls /opt/hadoop/bin/hdfs"
ssh worker2 "ls /opt/hadoop/bin/hdfs"
```

Kalau file `hdfs` dan `yarn` muncul → siap jalan.

---

#### 6. Membuat Direktori Data

Buat direktori untuk penyimpanan HDFS:

```bash
sudo mkdir -p /opt/hadoop_data/hdfs/namenode
sudo mkdir -p /opt/hadoop_data/hdfs/datanode
sudo chown -R insightera:insightera /opt/hadoop_data
```

Lakukan pada seluruh VM




---

#### 7. Inisialisasi dan Menjalankan HDFS

Langkah-langkah ini hanya dilakukan di **VM-Master**:

```bash
hdfs namenode -format
start-dfs.sh
```

Verifikasi proses berjalan:

```bash
jps
```

Output di Master:

```
NameNode
DataNode
SecondaryNameNode
```

Output di Worker:

```
DataNode
```

Uji status cluster:

```bash
hdfs dfsadmin -report
```

Jika muncul laporan dengan dua DataNode aktif dan kapasitas terdeteksi, berarti HDFS sudah berfungsi.

---

#### 8. Pengujian Dasar HDFS

Lakukan pengujian sederhana di Master:

```bash
echo "insightera test file" > test.txt
hdfs dfs -mkdir /user/insightera
hdfs dfs -put test.txt /user/insightera/
hdfs dfs -ls /user/insightera/
hdfs dfs -cat /user/insightera/test.txt
```

Jika file berhasil dibaca kembali dari HDFS, berarti sistem penyimpanan terdistribusi berfungsi normal.

---

#### 9. Validasi Web UI

Akses antarmuka web Hadoop dari browser:

* **NameNode UI:** `http://<PublicIP_Master>:9870`
* **SecondaryNameNode UI:** `http://<PublicIP_Master>:9868`

Pastikan halaman NameNode menampilkan dua DataNode aktif (worker1 dan worker2).

---

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



#### 10. Hasil Akhir Sprint 1 Bagian 3a

| Komponen            | Status                                | Keterangan                                   |
| ------------------- | ------------------------------------- | -------------------------------------------- |
| HDFS                | Aktif                                 | Sistem file terdistribusi berjalan di 3 node |
| DataNode            | 2 node aktif                          | worker1 dan worker2 terhubung                |
| Resource Management | Siap untuk konfigurasi YARN dan Spark | Hadoop environment sudah stabil              |

Cluster Hadoop kini berfungsi penuh untuk menyimpan dan membaca data secara terdistribusi, serta menjadi dasar integrasi dengan Spark pada bagian berikutnya.

Tahapan selanjutnya akan dilanjutkan pada **Sprint 1 Bagian 3b: Instalasi dan Konfigurasi Apache Spark 4.0.1 di seluruh node cluster.**