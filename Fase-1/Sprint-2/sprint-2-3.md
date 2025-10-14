### Sprint 2 – Bagian 3

**Konfigurasi Hadoop untuk Akses ADLS dan Implementasi Hybrid Mount (abfs://)**

Bagian ini merupakan tahap utama dalam proses integrasi Hadoop dengan Azure Data Lake Storage Gen2 (ADLS Gen2). Tujuannya adalah mengaktifkan koneksi dua arah antara **HDFS lokal** di cluster INSIGHTERA dan **penyimpanan cloud ADLS** menggunakan protokol `abfs://`.
Dengan konfigurasi ini, sistem dapat melakukan *read-write* data secara langsung ke container `bronze`, `silver`, dan `gold` di cloud tanpa proses pemindahan manual.

---

#### 1. Tujuan Bagian Ini

1. Mengaktifkan koneksi `abfs://` antara Hadoop dan ADLS Gen2 menggunakan OAuth2.
2. Menyesuaikan konfigurasi `core-site.xml` dan `hdfs-site.xml` agar mendukung hybrid storage.
3. Membuat direktori gabungan (hybrid mount point) untuk tiap zona (bronze, silver, gold).
4. Menguji proses baca-tulis data antar dua sistem (HDFS ↔ ADLS).
5. Menyiapkan skema sinkronisasi awal untuk pipeline data lakehouse INSIGHTERA.

---

#### 2. Konfigurasi Hadoop untuk Akses ADLS

Buka file konfigurasi di node Master:
`/opt/hadoop/etc/hadoop/core-site.xml`

Tambahkan atau pastikan parameter berikut sudah tersedia (melanjutkan konfigurasi Sprint 2 Bagian 2):

```xml
<configuration>
  <!-- Azure AD OAuth2 Configuration -->
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

  <!-- Default FileSystem Configuration -->
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://master:9000</value>
  </property>
</configuration>
```

Kemudian buka file `/opt/hadoop/etc/hadoop/hdfs-site.xml` dan tambahkan pengaturan berikut:

```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>2</value>
  </property>

  <property>
    <name>dfs.permissions.enabled</name>
    <value>true</value>
  </property>

  <property>
    <name>dfs.client.use.datanode.hostname</name>
    <value>true</value>
  </property>
</configuration>
```

Salin file ini ke semua node Worker:

```bash
scp /opt/hadoop/etc/hadoop/core-site.xml worker1:/opt/hadoop/etc/hadoop/
scp /opt/hadoop/etc/hadoop/core-site.xml worker2:/opt/hadoop/etc/hadoop/
scp /opt/hadoop/etc/hadoop/hdfs-site.xml worker1:/opt/hadoop/etc/hadoop/
scp /opt/hadoop/etc/hadoop/hdfs-site.xml worker2:/opt/hadoop/etc/hadoop/
```

---

#### 3. Uji Koneksi Langsung ke ADLS

Uji akses container `bronze` menggunakan perintah:

```bash
hdfs dfs -ls abfs://bronze@insighteradata.dfs.core.windows.net/
```

Jika konfigurasi benar, perintah akan menampilkan daftar direktori 15 unit kerja yang telah dibuat pada Sprint 2 Bagian 1.
Contoh hasil:

```
Found 15 items
drwxr-xr-x   -  akademik
drwxr-xr-x   -  keuangan
drwxr-xr-x   -  kemahasiswaan
...
```

---

#### 4. Pembuatan Direktori Hybrid Mount di HDFS

Agar pengguna Hadoop dapat mengakses ADLS melalui namespace lokal, buat direktori penghubung di HDFS:

```bash
hdfs dfs -mkdir /insightera
hdfs dfs -mkdir /insightera/bronze
hdfs dfs -mkdir /insightera/silver
hdfs dfs -mkdir /insightera/gold
```

Kemudian buat simbolik hybrid mount dengan *distcp* dan referensi abfs:

```bash
hadoop distcp abfs://bronze@insighteradata.dfs.core.windows.net/ /insightera/bronze
hadoop distcp abfs://silver@insighteradata.dfs.core.windows.net/ /insightera/silver
hadoop distcp abfs://gold@insighteradata.dfs.core.windows.net/ /insightera/gold
```

Verifikasi hasil:

```bash
hdfs dfs -ls /insightera/bronze
```

Jika hasilnya menampilkan folder 15 unit kerja, berarti koneksi hybrid mount telah berfungsi.

---

#### 5. Uji Tulis dan Baca Data

Uji upload file dari HDFS lokal ke ADLS:

```bash
echo "data hybrid sync test" > sync-test.txt
hdfs dfs -put sync-test.txt abfs://bronze@insighteradata.dfs.core.windows.net/fsains/
```

Periksa hasil di portal Azure Storage Explorer pada container `bronze/fsains`.

Kemudian uji baca dari ADLS:

```bash
hdfs dfs -cat abfs://bronze@insighteradata.dfs.core.windows.net/fsains/sync-test.txt
```

Jika isi file tampil dengan benar, integrasi dua arah sudah berhasil.

---

#### 6. Uji Bandwidth dan Performa Transfer

Gunakan perintah berikut untuk menguji kecepatan transfer antara HDFS dan ADLS:

```bash
time hadoop distcp /user/insightera/test_dataset \
abfs://bronze@insighteradata.dfs.core.windows.net/keuangan/
```

Catat hasil waktu eksekusi dan bandwidth rata-rata yang muncul di output.
Biasanya transfer data 1 GB pada koneksi Azure regional membutuhkan ±15–25 detik untuk cluster 3 node (developer mode).

---

#### 7. Penambahan Tag dan Monitoring Hybrid

Tambahkan label untuk memudahkan identifikasi sumber data hybrid:

```bash
az storage account update \
  --name insighteradata \
  --resource-group RG-Datalakehouse-Insightera \
  --set tags.architecture="hybrid" tags.sync="enabled"
```

Cek aktivitas di portal Azure → *Monitoring → Activity Log* untuk memastikan transfer `distcp` dan upload ADLS terekam.

---

#### 8. Struktur Hybrid Data Lakehouse yang Terbentuk

Setelah konfigurasi ini, struktur penyimpanan INSIGHTERA menjadi sebagai berikut:

```
HDFS (/opt/hadoop_data/)
└── insightera/
    ├── bronze/ → abfs://bronze@insighteradata.dfs.core.windows.net/
    ├── silver/ → abfs://silver@insighteradata.dfs.core.windows.net/
    └── gold/   → abfs://gold@insighteradata.dfs.core.windows.net/

ADLS Gen2 (insighteradata.dfs.core.windows.net)
├── bronze/
│   ├── akademik/
│   ├── keuangan/
│   └── ...
├── silver/
│   ├── akademik/
│   ├── keuangan/
│   └── ...
└── gold/
    ├── akademik/
    ├── keuangan/
    └── ...
```

Dengan struktur ini, seluruh proses ingest, cleansing, dan analitik dapat dilakukan dari HDFS maupun langsung ke ADLS tanpa duplikasi data.

---

#### 9. Hasil Akhir Sprint 2 Bagian 3

| Komponen           | Status    | Keterangan                                    |
| ------------------ | --------- | --------------------------------------------- |
| Autentikasi OAuth2 | Aktif     | Menggunakan Service Principal `sp-insightera` |
| Protokol abfs://   | Berfungsi | Hadoop terhubung ke ADLS Gen2                 |
| Hybrid Mount       | Berhasil  | Sinkronisasi dua arah HDFS ↔ ADLS             |
| Data Lakehouse     | Aktif     | Struktur Bronze/Silver/Gold terpasang         |
| Uji Baca/Tulis     | Sukses    | File dapat diakses dari dua sisi              |

---

#### 10. Kesimpulan

Konfigurasi ini berhasil menghubungkan Hadoop Cluster lokal INSIGHTERA dengan Azure Data Lake Storage Gen2 menggunakan driver `abfs://`.
Kini HDFS dapat berfungsi sebagai *on-premise caching layer* sementara ADLS berperan sebagai *cloud persistent layer*, membentuk arsitektur **Hybrid Data Lakehouse** yang siap digunakan untuk pipeline data ingestion dan federasi data mart.

Langkah selanjutnya akan dilanjutkan pada **Sprint 2 – Bagian 4: Pengujian Sinkronisasi dan Validasi Kinerja Hybrid Data Lakehouse**, untuk memastikan performa, latency, dan konsistensi data antara penyimpanan lokal dan cloud.
