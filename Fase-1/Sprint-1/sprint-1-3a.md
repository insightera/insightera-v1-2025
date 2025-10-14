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
ssh -i ~/.ssh/insightera_dev_key insightera@<PublicIP_VM>
```

Kemudian, pada setiap VM jalankan perintah berikut untuk menyiapkan direktori kerja Hadoop:

```bash
sudo apt update -y
sudo apt install openjdk-11-jdk -y
```

Verifikasi Java:

```bash
java -version
```

Output yang diharapkan:

```
openjdk version "11.0.xx"
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
scp -r /opt/hadoop/etc/hadoop/ worker1:/opt/hadoop/etc/
scp -r /opt/hadoop/etc/hadoop/ worker2:/opt/hadoop/etc/
```

---

#### 6. Membuat Direktori Data

Buat direktori untuk penyimpanan HDFS:

```bash
sudo mkdir -p /opt/hadoop_data/hdfs/namenode
sudo mkdir -p /opt/hadoop_data/hdfs/datanode
sudo chown -R insightera:insightera /opt/hadoop_data
```

Lakukan pada seluruh node.

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

#### 10. Hasil Akhir Sprint 1 Bagian 3a

| Komponen            | Status                                | Keterangan                                   |
| ------------------- | ------------------------------------- | -------------------------------------------- |
| HDFS                | Aktif                                 | Sistem file terdistribusi berjalan di 3 node |
| DataNode            | 2 node aktif                          | worker1 dan worker2 terhubung                |
| Resource Management | Siap untuk konfigurasi YARN dan Spark | Hadoop environment sudah stabil              |

Cluster Hadoop kini berfungsi penuh untuk menyimpan dan membaca data secara terdistribusi, serta menjadi dasar integrasi dengan Spark pada bagian berikutnya.

Tahapan selanjutnya akan dilanjutkan pada **Sprint 1 Bagian 3b: Instalasi dan Konfigurasi Apache Spark 4.0.1 di seluruh node cluster.**