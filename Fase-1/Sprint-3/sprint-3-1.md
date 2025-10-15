### Sprint 3 – Bagian 1

**Persiapan dan Instalasi Hive Metastore (PostgreSQL Backend) – Versi Hive 4.1.0**

Bagian ini adalah tahap awal dalam integrasi metadata INSIGHTERA yang berfungsi membangun sistem **Hive Metastore** menggunakan **PostgreSQL** sebagai *metadata backend database*.
Hive Metastore akan menjadi pusat pengelolaan metadata untuk seluruh data mart dan lapisan Medallion Architecture (Bronze–Silver–Gold), baik di HDFS lokal maupun ADLS Gen2.

Versi yang digunakan adalah **Apache Hive 4.1.0**, yang mendukung integrasi native dengan Hadoop 3.4.x dan Spark 4.0.x serta sudah stabil untuk penggunaan hybrid environment.

---

#### 1. Tujuan Bagian Ini

1. Menginstal dan mengonfigurasi **PostgreSQL** sebagai database Hive Metastore.
2. Membuat database `hive_metastore` dan user `hive_user`.
3. Menginstal Apache Hive 4.1.0 pada VM Management dan VM Master.
4. Mengonfigurasi koneksi JDBC Hive–PostgreSQL di `hive-site.xml`.
5. Melakukan inisialisasi schema Metastore dan pengujian koneksi.

---

#### 2. Persiapan Lingkungan dan Prasyarat

Jalankan langkah-langkah berikut pada **VM-Management** (tempat Hive dan PostgreSQL diinstal):

```bash
sudo apt update -y
sudo apt install openjdk-11-jdk wget tar postgresql postgresql-contrib -y
```

Pastikan Java telah terinstal:

```bash
java -version
```

Output yang diharapkan:

```
openjdk version "11.0.xx"
```

---

#### 3. Instalasi dan Konfigurasi PostgreSQL

Masuk ke shell PostgreSQL:

```bash
sudo -u postgres psql
```

Buat database dan user untuk Hive:

```sql
CREATE DATABASE hive_metastore;
CREATE USER hive_user WITH PASSWORD 'hive_password';
ALTER ROLE hive_user SET client_encoding TO 'utf8';
ALTER ROLE hive_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE hive_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE hive_metastore TO hive_user;
\q
```

Verifikasi:

```bash
sudo -u postgres psql -c "\l" | grep hive_metastore
```

---

#### 4. Unduh dan Instal Apache Hive 4.1.0

Masih di VM-Management:

```bash
cd /opt
sudo wget https://downloads.apache.org/hive/hive-4.1.0/apache-hive-4.1.0-bin.tar.gz
sudo tar -xzf apache-hive-4.1.0-bin.tar.gz
sudo mv apache-hive-4.1.0-bin hive
sudo chown -R insightera:insightera /opt/hive
```

Tambahkan variabel environment di `.bashrc`:

```bash
echo "export HIVE_HOME=/opt/hive" >> ~/.bashrc
echo "export PATH=\$PATH:\$HIVE_HOME/bin:\$HIVE_HOME/sbin" >> ~/.bashrc
echo "export HADOOP_HOME=/opt/hadoop" >> ~/.bashrc
echo "export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64" >> ~/.bashrc
source ~/.bashrc
```

Verifikasi instalasi:

```bash
hive --version
```

Output:

```
Hive 4.1.0
Subversion git://... 
Compiled by ...
```

---

#### 5. Konfigurasi Hive Metastore untuk PostgreSQL

Masuk ke direktori konfigurasi:

```bash
cd $HIVE_HOME/conf
cp hive-default.xml.template hive-site.xml
```

Edit file `hive-site.xml` dengan konfigurasi berikut:

```xml
<configuration>

  <!-- Metastore Connection -->
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://localhost:5432/hive_metastore</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive_user</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>hive_password</value>
  </property>

  <property>
    <name>datanucleus.schema.autoCreateAll</name>
    <value>true</value>
  </property>

  <!-- Hive Metastore Service -->
  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>

  <property>
    <name>hive.metastore.uris</name>
    <value>thrift://localhost:9083</value>
  </property>

  <!-- Hadoop Integration -->
  <property>
    <name>hive.exec.scratchdir</name>
    <value>/tmp/hive</value>
  </property>

  <property>
    <name>hive.metastore.schema.verification</name>
    <value>false</value>
  </property>

</configuration>
```

---

#### 6. Menambahkan JDBC Driver PostgreSQL ke Hive

Unduh driver JDBC untuk PostgreSQL:

```bash
cd $HIVE_HOME/lib
wget https://jdbc.postgresql.org/download/postgresql-42.7.3.jar
```

Verifikasi file:

```bash
ls postgresql-*.jar
```

---

#### 7. Inisialisasi Hive Metastore Schema

Jalankan inisialisasi schema menggunakan perintah bawaan Hive:

```bash
schematool -dbType postgres -initSchema -userName hive_user -passWord hive_password
```

Jika berhasil, output akan menampilkan:

```
Hive schema tool.
Starting metastore schema initialization to version 4.1.0
Initialization script completed
```

---

#### 8. Menjalankan Layanan Hive Metastore

Mulai layanan Metastore:

```bash
hive --service metastore &
```

Verifikasi bahwa port `9083` sudah aktif:

```bash
netstat -tulnp | grep 9083
```

Output:

```
tcp6   0   0 :::9083   :::*   LISTEN   29873/java
```

---

#### 9. Pengujian Koneksi dari Hive CLI

Dari VM-Management atau Master, jalankan:

```bash
hive
```

Lalu di prompt Hive:

```sql
SHOW DATABASES;
```

Hasil awal:

```
default
```

Buat database percobaan:

```sql
CREATE DATABASE testdb;
SHOW DATABASES;
```

Jika `testdb` muncul dalam daftar, koneksi ke PostgreSQL Metastore telah berfungsi.

---

#### 10. Pengujian Metastore dari SparkSQL

Untuk memastikan Spark juga dapat menggunakan metadata Hive, jalankan di VM-Master:

```bash
spark-sql --master yarn --conf spark.sql.catalogImplementation=hive -e "SHOW DATABASES;"
```

Jika hasil yang sama (`default`, `testdb`) muncul, berarti Spark berhasil membaca Metastore Hive yang sama.

---

#### 11. Hasil Akhir Sprint 3 Bagian 1

| Komponen               | Status          | Keterangan                         |
| ---------------------- | --------------- | ---------------------------------- |
| PostgreSQL Server      | Aktif           | Database `hive_metastore` berjalan |
| User hive_user         | Dibuat          | Hak akses penuh ke database        |
| Hive 4.1.0             | Terinstal       | Berfungsi di VM-Management         |
| JDBC Driver PostgreSQL | Ditambahkan     | Terhubung dengan Hive              |
| Schema Metastore       | Terinisialisasi | Menggunakan `schematool`           |
| Hive Metastore Service | Aktif           | Berjalan di port 9083              |
| Uji CLI Hive           | Sukses          | Database `testdb` berhasil dibuat  |
| Integrasi SparkSQL     | Sukses          | Spark membaca metadata Hive        |

---

#### 12. Kesimpulan

Tahapan ini berhasil membangun **Hive Metastore versi 4.1.0** dengan backend **PostgreSQL** yang stabil dan kompatibel dengan Hadoop–Spark INSIGHTERA.
Metastore kini menjadi *single source of truth* untuk seluruh metadata sistem dan siap digunakan untuk membangun struktur database federatif pada 15 data mart INSIGHTERA.

Langkah selanjutnya akan dilanjutkan pada **Sprint 3 – Bagian 2: Instalasi Apache Hive 4.1.0 pada Cluster dan Integrasi dengan HDFS–ADLS Hybrid**.
