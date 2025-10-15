💡 Sangat bagus kamu sudah sampai tahap uji tulis ke HDFS — dan error yang muncul ini:

```
put: `/user/insightera/': No such file or directory: `hdfs://master:9000/user/insightera'
ls: `/user/insightera/': No such file or directory
```

artinya sederhana:
👉 **HDFS belum dibuatkan direktori `/user/insightera` di dalam namespace HDFS.**

HDFS tidak otomatis membuat folder home untuk setiap user — kita harus buat manual setelah NameNode diformat dan service `dfs` sudah berjalan.

---

## ✅ **Langkah Perbaikan**

### **1️⃣ Pastikan Hadoop cluster aktif**

Pastikan service DFS jalan di master:

```bash
jps
```

Harus ada proses:

```
NameNode
DataNode
SecondaryNameNode
```

Kalau belum ada, start dulu:

```bash
start-dfs.sh
```

---

### **2️⃣ Buat folder user di HDFS**

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

### **3️⃣ Ulangi uji coba upload**

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

---

## 🧭 **Kenapa ini terjadi**

* Direktori `/user/<username>` **tidak otomatis dibuat** setelah format HDFS.
* Saat kamu jalankan `hdfs dfs -put`, HDFS akan menolak kalau path tujuan tidak ada.

---

## 💡 **Tips lanjutan**

Untuk membuat folder otomatis setiap kali cluster baru diformat (praktik umum di produksi), kamu bisa tambahkan di `bootstrap.sh`:

```bash
hdfs dfs -mkdir -p /user/insightera
hdfs dfs -chown -R insightera /user/insightera
```

atau gunakan folder `input`, `output`, `staging`, `warehouse`:

```bash
hdfs dfs -mkdir -p /data/{bronze,silver,gold}
hdfs dfs -chmod -R 755 /data
```

---

Kalau kamu mau, aku bisa bantu buatkan **skrip inisialisasi HDFS (`init-hdfs-structure.sh`)**
yang langsung menyiapkan:

* `/user/insightera`
* `/data/bronze`, `/data/silver`, `/data/gold`
* `/warehouse/hive`
  Lengkap dengan `chown` dan permission yang sesuai.
  Apakah mau saya buatkan sekalian?
