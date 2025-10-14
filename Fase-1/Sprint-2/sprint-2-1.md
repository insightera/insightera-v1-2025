# Sprint 2 – Bagian 1

**Persiapan Azure Data Lake Storage Gen2 dan Struktur Folder Data Mart INSIGHTERA**

Tahapan ini bertujuan membangun komponen penyimpanan cloud pada arsitektur hybrid INSIGHTERA menggunakan **Azure Data Lake Storage Gen2 (ADLS Gen2)**. Layanan ini akan menjadi lapisan penyimpanan utama yang terhubung secara langsung ke HDFS di cluster Hadoop–Spark yang telah dikonfigurasi pada Sprint 1.

Selain membangun tiga zona utama penyimpanan (Bronze, Silver, Gold), tahap ini juga menyiapkan struktur folder berdasarkan **15 unit kerja ITERA** yang akan memiliki data mart masing-masing.

---

## 1. Tujuan Bagian Ini

1. Membuat akun penyimpanan `insighteradata` dengan fitur *Hierarchical Namespace* aktif.
2. Membuat tiga container utama: `bronze`, `silver`, dan `gold`.
3. Menyusun struktur direktori sesuai dengan 15 unit kerja/data mart INSIGHTERA.
4. Mengatur kebijakan keamanan dasar untuk setiap container.
5. Memverifikasi konektivitas awal dari terminal lokal dan portal Azure.

---

## 2. Pembuatan Storage Account ADLS Gen2

Jalankan perintah berikut di terminal lokal (macOS/Linux) setelah login ke Azure CLI:

```bash
az storage account create \
  --name insighteradata \
  --resource-group RG-Datalakehouse-Insightera \
  --location southeastasia \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true \
  --https-only true
```

Verifikasi hasil:

```bash
az storage account show \
  --name insighteradata \
  --resource-group RG-Datalakehouse-Insightera \
  --query "isHnsEnabled"
```

Jika nilai yang ditampilkan `true`, maka fitur *Hierarchical Namespace* sudah aktif.

---

## 3. Pembuatan Container Medallion Architecture

INSIGHTERA menggunakan pendekatan **Medallion Architecture** dengan tiga zona utama penyimpanan:

* **Bronze**: menyimpan data mentah dari sumber eksternal dan sistem operasional.
* **Silver**: menyimpan data yang telah dibersihkan dan distandardisasi.
* **Gold**: menyimpan data teragregasi dan siap analisis (untuk dashboard dan EDW).

Buat container dengan perintah berikut:

```bash
az storage container create --name bronze --account-name insighteradata
az storage container create --name silver --account-name insighteradata
az storage container create --name gold --account-name insighteradata
```

Verifikasi container:

```bash
az storage container list --account-name insighteradata -o table
```

---

## 4. Struktur Folder Berdasarkan 15 Unit Kerja ITERA

Setiap container (`bronze`, `silver`, `gold`) akan memiliki 15 folder yang mewakili unit kerja/data mart.
Struktur direktori yang disiapkan adalah sebagai berikut:

| No | Unit Kerja / Domain                  | Folder           |
| -- | ------------------------------------ | ---------------- |
| 1  | Akademik                             | `/akademik`      |
| 2  | Non Akademik                         | `/nonakademik`   |
| 3  | Kemahasiswaan                        | `/kemahasiswaan` |
| 4  | LPPM                                 | `/lppm`          |
| 5  | LPMPP                                | `/lpmp`          |
| 6  | Keuangan                             | `/keuangan`      |
| 7  | Kepegawaian                          | `/kepegawaian`   |
| 8  | Keamanan Siber                       | `/keamanansiber` |
| 9  | Satu Data ITERA                      | `/satudata`      |
| 10 | Kebun Raya ITERA                     | `/kebunraya`     |
| 11 | Fakultas Sains                       | `/fsains`        |
| 12 | Fakultas Teknologi Industri          | `/fti`           |
| 13 | Fakultas Infrastruktur & Kewilayahan | `/ftik`           |
| 14 | Sarana & Prasarana                   | `/sarpras`       |
| 15 | K3L                                  | `/k3l`           |

Buat semua direktori ini pada masing-masing container.

Contoh untuk container `bronze`:

```bash
az storage fs directory create --account-name insighteradata --file-system bronze --name akademik
az storage fs directory create --account-name insighteradata --file-system bronze --name nonakademik
az storage fs directory create --account-name insighteradata --file-system bronze --name kemahasiswaan
az storage fs directory create --account-name insighteradata --file-system bronze --name lppm
az storage fs directory create --account-name insighteradata --file-system bronze --name lpmp
az storage fs directory create --account-name insighteradata --file-system bronze --name keuangan
az storage fs directory create --account-name insighteradata --file-system bronze --name kepegawaian
az storage fs directory create --account-name insighteradata --file-system bronze --name keamanansiber
az storage fs directory create --account-name insighteradata --file-system bronze --name satudata
az storage fs directory create --account-name insighteradata --file-system bronze --name kebunraya
az storage fs directory create --account-name insighteradata --file-system bronze --name fsains
az storage fs directory create --account-name insighteradata --file-system bronze --name fti
az storage fs directory create --account-name insighteradata --file-system bronze --name ftik
az storage fs directory create --account-name insighteradata --file-system bronze --name sarpras
az storage fs directory create --account-name insighteradata --file-system bronze --name k3l
```

Langkah yang sama dilakukan untuk container `silver` dan `gold`.

---

## 5. Penetapan Struktur Medallion dan Hak Akses

Gunakan konvensi berikut agar setiap folder di tiga zona memiliki kesamaan struktur:

```
bronze/
  ├─ akademik/
  ├─ keuangan/
  ├─ kemahasiswaan/
  ├─ ...
silver/
  ├─ akademik/
  ├─ keuangan/
  ├─ ...
gold/
  ├─ akademik/
  ├─ keuangan/
  ├─ ...
```

Tambahkan label/tag manajemen:

```bash
az tag create --resource-id $(az storage account show --name insighteradata --query id -o tsv) \
  --tags project=insightera env=production architecture=hybrid phase=sprint2
```

---

## 6. Pengujian Akses Dasar

Uji koneksi ke Storage Account:

```bash
az storage fs list --account-name insighteradata -o table
```

Uji pembuatan file percobaan di folder bronze:

```bash
echo "test hybrid data lakehouse" > test.txt
az storage fs file upload --account-name insighteradata --file-system bronze --path akademik/test.txt --source test.txt
```

Verifikasi file berhasil diunggah melalui portal Azure atau Storage Explorer.

---

## 7. Hasil Akhir Sprint 2 Bagian 1

| Komponen        | Nama / Nilai                      | Status | Keterangan                      |
| --------------- | --------------------------------- | ------ | ------------------------------- |
| Storage Account | `insighteradata`                  | Aktif  | HNS diaktifkan                  |
| Container       | `bronze`, `silver`, `gold`        | Dibuat | Struktur sesuai Medallion       |
| Folder          | 15 unit kerja ITERA               | Dibuat | Struktur konsisten di tiga zona |
| Tag Manajemen   | project=insightera, phase=sprint2 | Aktif  | Untuk tracking resource         |
| Uji Upload      | test.txt di `/bronze/akademik/`   | Sukses | Koneksi berfungsi               |

---

## 8. Kesimpulan

Tahapan ini berhasil membangun lapisan penyimpanan cloud ADLS Gen2 sebagai fondasi *Hybrid Data Lakehouse* INSIGHTERA.
Seluruh container dan struktur folder telah disiapkan untuk 15 unit kerja ITERA sehingga dapat digunakan untuk proses integrasi HDFS dan sinkronisasi hybrid di bagian berikutnya.

Langkah selanjutnya akan dilanjutkan pada **Sprint 2 – Bagian 2: Pembuatan Service Principal dan Konfigurasi Autentikasi OAuth2 untuk Hadoop–ADLS.**
