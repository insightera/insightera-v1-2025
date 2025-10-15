### Sprint 3 – Bagian 2

**Instalasi Apache Hive 4.1.0 pada Cluster dan Integrasi dengan HDFS–ADLS Hybrid**

Bagian ini melanjutkan hasil dari Sprint 3 Bagian 1, di mana Hive Metastore berbasis PostgreSQL telah aktif di VM Management.
Tahapan ini bertujuan mengintegrasikan **Apache Hive 4.1.0** dengan **Hadoop–ADLS hybrid cluster**, sehingga Hive dapat membaca dan menulis langsung ke HDFS serta Azure Data Lake Storage Gen2 (ADLS Gen2) melalui protokol `abfs://`.

Setelah konfigurasi ini selesai, seluruh node dalam cluster INSIGHTERA dapat menggunakan Hive untuk manajemen metadata, query SQL terdistribusi, serta federasi awal antar data mart.

---

#### 1. Tujuan Bagian Ini

1. Menginstal dan mengonfigurasi Hive 4.1.0 di VM Master dan Worker.
2. Mengintegrasikan Hive dengan Hadoop (HDFS + YARN) dan ADLS Gen2.
3. Mengatur koneksi Metastore Hive ke database PostgreSQL yang telah dibuat.
4. Menguji fungsionalitas Hive CLI dan HiveServer2.
5. Memastikan Hive dapat mengakses direktori hybrid (`abfs://`) pada ADLS.

---

#### 2. Persiapan Instalasi di Node Master

Lakukan langkah berikut pada VM Master:

```bash
sudo apt update -y
sudo apt install openjdk-11-jdk wget tar -y
```

Unduh Hive 4.1.0:

```bash
cd /opt
sudo wget https://downloads.apache.org/hive/hive-4.1.0/apache-hive-4.1.0-bin.tar.gz
sudo tar -xzf apache-hive-4.1.0-bin.tar.gz
sudo mv apache-hive-4.1.0-bin hive
sudo chown -R insightera:insightera /opt/hive
```

Tambahkan variabel lingkungan ke `.bashrc`:

```bash
echo "export HIVE_HOME=/opt/hive" >> ~/.bashrc
echo "export PATH=\$PATH:\$HIVE_HOME/bin:\$HIVE_HOME/sbin" >> ~/.bashrc
echo "export HADOOP_HOME=/opt/hadoop" >> ~/.bashrc
echo "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64" >> ~/.bashrc
source ~/.bashrc
```

Verifikasi versi:

```bash
hive --version
```

Hasil:

```
Hive 4.1.0
Subversion git://...
```

---

#### 3. Menyalin Konfigurasi Hadoop dan Metastore

Agar Hive dapat terhubung ke Hadoop dan PostgreSQL, salin file konfigurasi dari VM Management ke VM Master:

```bash
scp -r insightera@management:/opt/hadoop/etc/hadoop/core-site.xml /opt/hadoop/etc/hadoop/
scp -r insightera@management:/opt/hadoop/etc/hadoop/hdfs-site.xml /opt/hadoop/etc/hadoop/
scp -r insightera@management:/opt/hive/conf/hive-site.xml /opt/hive/conf/
scp -r insightera@management:/opt/hive/lib/postgresql-42.7.3.jar /opt/hive/lib/
```

Pastikan direktori `/opt/hive/conf` berisi file berikut:

```
hive-site.xml
hive-env.sh
hive-log4j2.properties
```

---

#### 4. Konfigurasi File `hive-env.sh`

Buka file:

```bash
cd /opt/hive/conf
cp hive-env.sh.template hive-env.sh
nano hive-env.sh
```

Tambahkan konfigurasi berikut:

```bash
export HADOOP_HOME=/opt/hadoop
export HIVE_CONF_DIR=/opt/hive/conf
export HIVE_AUX_JARS_PATH=/opt/hive/lib
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export HIVE_HOME=/opt/hive
```

---

#### 5. Konfigurasi Integrasi HDFS dan ADLS

Pastikan file `core-site.xml` telah berisi konfigurasi `abfs://` seperti yang dibuat di Sprint 2 Bagian 3, misalnya:

```xml
<property>
  <name>fs.azure.account.auth.type.insighteradata.dfs.core.windows.net</name>
  <value>OAuth</value>
</property>
<property>
  <name>fs.azure.account.oauth.provider.type.insighteradata.dfs.core.windows.net</name>
  <value>org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider</value>
</property>
<property>
  <name>fs.azure.account.oauth2.client.id.insighteradata.dfs.core.windows.net</name>
  <value>xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx</value>
</property>
<property>
  <name>fs.azure.account.oauth2.client.secret.insighteradata.dfs.core.windows.net</name>
  <value>yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy</value>
</property>
<property>
  <name>fs.azure.account.oauth2.client.endpoint.insighteradata.dfs.core.windows.net</name>
  <value>https://login.microsoftonline.com/aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa/oauth2/token</value>
</property>
```

Tambahkan konfigurasi direktori warehouse di HDFS:

```xml
<property>
  <name>hive.metastore.warehouse.dir</name>
  <value>abfs://silver@insighteradata.dfs.core.windows.net/hive/warehouse</value>
</property>
```

---

#### 6. Menjalankan Hive Metastore dan HiveServer2

Jalankan dari VM Management:

```bash
nohup hive --service metastore > metastore.log 2>&1 &
```

Kemudian di VM Master jalankan HiveServer2:

```bash
nohup hive --service hiveserver2 > hiveserver2.log 2>&1 &
```

Verifikasi port:

```bash
netstat -tulnp | grep -E "9083|10000"
```

Hasil:

```
tcp6   0   0 :::9083   :::*   LISTEN   28573/java   (Metastore)
tcp6   0   0 :::10000  :::*   LISTEN   28605/java   (HiveServer2)
```

---

#### 7. Pengujian Hive CLI dan Query Awal

Masuk ke Hive CLI:

```bash
hive
```

Lakukan pengujian:

```sql
SHOW DATABASES;
CREATE DATABASE test_integration LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/test_integration';
SHOW DATABASES;
```

Periksa hasil di portal Azure:

```
Container: silver → Folder: /test_integration
```

Buat tabel sederhana untuk uji tulis:

```sql
USE test_integration;
CREATE TABLE sample_data (id INT, name STRING)
STORED AS PARQUET
LOCATION 'abfs://silver@insighteradata.dfs.core.windows.net/test_integration/sample_data';

INSERT INTO sample_data VALUES (1, 'Hybrid Test'), (2, 'ADLS Hive');
SELECT * FROM sample_data;
```

Jika hasil query menampilkan dua baris data, maka integrasi berhasil.

---

#### 8. Pengujian HiveServer2 via Beeline

Dari VM Master atau lokal (jika port terbuka):

```bash
beeline -u jdbc:hive2://master:10000 -n insightera -p ""
```

Jalankan query:

```sql
SHOW DATABASES;
SELECT COUNT(*) FROM test_integration.sample_data;
```

Jika hasil muncul, berarti koneksi ke HiveServer2 dan Metastore PostgreSQL berjalan dengan baik.

---

#### 9. Validasi Integrasi ke ADLS (Hybrid Layer)

Uji direktori hasil penyimpanan tabel Hive di Azure Storage:

```
/silver/test_integration/sample_data/
```

File Parquet hasil insert akan muncul di dalam container `silver` dengan format:

```
part-00000-xxxx.snappy.parquet
```

Ini menunjukkan bahwa Hive berhasil menulis data secara langsung ke ADLS melalui driver `abfs://`.

---

#### 10. Hasil Akhir Sprint 3 Bagian 2

| Komponen          | Status   | Keterangan                            |
| ----------------- | -------- | ------------------------------------- |
| Hive 4.1.0        | Aktif    | Terinstal di VM Master dan Management |
| Hive Metastore    | Aktif    | Menggunakan PostgreSQL backend        |
| HiveServer2       | Berjalan | Port 10000 aktif                      |
| ADLS Integration  | Sukses   | Protokol abfs:// berfungsi            |
| Query SQL         | Berhasil | CREATE, INSERT, SELECT berfungsi      |
| Akses via Beeline | Berhasil | Koneksi JDBC stabil                   |

---

#### 11. Kesimpulan

Tahapan ini berhasil mengintegrasikan Apache Hive 4.1.0 dengan ekosistem Hadoop–Spark dan ADLS Gen2, menjadikan Hive sebagai lapisan metadata utama pada **Hybrid Data Lakehouse INSIGHTERA**.
Hive kini dapat melakukan operasi SQL langsung ke data yang tersimpan di HDFS maupun ADLS dengan backend Metastore PostgreSQL yang terpusat.

Langkah selanjutnya akan dilanjutkan pada **Sprint 3 – Bagian 3: Pembuatan Skema dan Database Hive untuk 15 Data Mart INSIGHTERA.**
