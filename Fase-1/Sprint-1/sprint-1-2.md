### Sprint 1 – Bagian 2

**Pembuatan Virtual Machine Cluster (Master, Management, Worker 1, dan Worker 2) – Ubuntu 24.04 LTS**

Bagian ini melanjutkan hasil konfigurasi jaringan dan resource group pada Sprint 1 Bagian 1. Tujuan dari bagian ini adalah membuat empat buah Virtual Machine (VM) yang akan berfungsi sebagai node utama dalam cluster hybrid INSIGHTERA. Keempat VM ini akan ditempatkan pada subnet `Subnet-Compute` di dalam VNet yang telah dibuat sebelumnya, dengan spesifikasi ringan untuk tahap pengembangan (developer cluster).

---

#### 1. Spesifikasi Virtual Machine

Semua VM menggunakan sistem operasi Ubuntu 24.04 LTS, dengan konfigurasi spesifikasi sebagai berikut:

| Node          | Peran                                        | vCPU | RAM  | Disk  | IP Privat |
| ------------- | -------------------------------------------- | ---- | ---- | ----- | --------- |
| VM-Master     | NameNode, ResourceManager, Spark Master      | 2    | 4 GB | 64 GB | 10.0.1.7  |
| VM-Management | Airflow, PostgreSQL, Hive Metastore, Grafana | 2    | 4 GB | 64 GB | 10.0.1.8  |
| VM-Worker1    | DataNode, NodeManager, Spark Worker          | 2    | 4 GB | 64 GB | 10.0.1.9  |
| VM-Worker2    | DataNode, NodeManager, Stream Gateway        | 2    | 4 GB | 64 GB | 10.0.1.10 |

Tipe VM yang direkomendasikan: `Standard_B2s`
Wilayah: `southeastasia`
Resource Group: `RG-Datalakehouse-Insightera`
VNet: `VNet-Datalakehouse-Insightera`
Subnet: `Subnet-Compute`

---

#### 2. Persiapan SSH Key (Lokal)

Sebelum membuat VM, buat terlebih dahulu pasangan kunci SSH di mesin lokal (macOS atau Linux):

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/insightera -N ""
```

Kunci publik `~/.ssh/insightera.pub` akan digunakan untuk otentikasi VM.

---

#### 3. Pembuatan Virtual Machine

Gunakan perintah berikut untuk membuat masing-masing node VM:

```bash
# VM-Master
az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Master \
  --image Ubuntu2404 \
  --size Standard_B2s \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --os-disk-size-gb 64 \
  --nsg NSG-Insightera \
  --private-ip-address 10.0.1.7 \
  --tags project=insightera role=master env=dev
```

```bash
# VM-Management
az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Management \
  --image Ubuntu2404 \
  --size Standard_B1ms \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --os-disk-size-gb 64 \
  --nsg NSG-Insightera \
  --private-ip-address 10.0.1.8 \
  --tags project=insightera role=management env=dev

```

```bash
# VM-Worker1
az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Worker1 \
  --image Ubuntu2404 \
  --size Standard_B1ms \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --os-disk-size-gb 64 \
  --nsg NSG-Insightera \
  --private-ip-address 10.0.1.9 \
  --tags project=insightera role=worker env=dev
```

```bash
# VM-Worker2
az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Worker2 \
  --image Ubuntu2404 \
  --size Standard_B1ms \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --os-disk-size-gb 64 \
  --nsg NSG-Insightera \
  --private-ip-address 10.0.1.10 \
  --tags project=insightera role=worker env=dev
```

Masing-masing VM akan otomatis dibuat dengan IP privat tetap sesuai konfigurasi. Azure juga akan menghasilkan IP publik sementara untuk koneksi SSH dari luar.

Kamu bisa cek kuota semua VM family:
```
az vm list-usage --location southeastasia --output table
```

---

#### 4. Verifikasi Hasil

Tampilkan seluruh IP publik dan privat dari VM yang telah dibuat:

```bash
az vm list-ip-addresses -g RG-Datalakehouse-Insightera -o table
```

Contoh hasil keluaran:

```
Name            PublicIP         PrivateIP
--------------  ----------------  ----------
VM-Master       20.55.241.7      10.0.1.7
VM-Management   20.55.241.8      10.0.1.8
VM-Worker1      20.55.241.9      10.0.1.9
VM-Worker2      20.55.241.10     10.0.1.10
```

---

#### 5. Uji Koneksi SSH

Gunakan kunci SSH yang telah dibuat untuk mengakses VM Master:

```bash
ssh -i ~/.ssh/insightera_dev_key insightera@<PublicIP_VM-Master>
```

Setelah berhasil masuk ke VM Master, uji koneksi antar-node:

```bash
ping -c 2 10.0.1.8
ping -c 2 10.0.1.9
ping -c 2 10.0.1.10
```

Jika semua node dapat saling berkomunikasi, jaringan internal cluster telah berfungsi dengan baik.

---

#### 6. Menambahkan Host Mapping

Agar setiap node dapat saling mengenali tanpa IP, tambahkan entri ke dalam file `/etc/hosts` pada masing-masing VM:

```
10.0.1.7 master
10.0.1.8 management
10.0.1.9 worker1
10.0.1.10 worker2
```

Gunakan perintah berikut di VM Master untuk mengedit:

```bash
sudo nano /etc/hosts
```

Simpan perubahan dengan `Ctrl + O` lalu keluar dengan `Ctrl + X`.

---

#### 7. Hasil Akhir Sprint 1 Bagian 2

| Node          | Nama Host  | Fungsi Utama                            | Status |
| ------------- | ---------- | --------------------------------------- | ------ |
| VM-Master     | master     | NameNode, ResourceManager, Spark Master | Aktif  |
| VM-Management | management | Airflow, PostgreSQL, Metastore          | Aktif  |
| VM-Worker1    | worker1    | DataNode, Spark Worker                  | Aktif  |
| VM-Worker2    | worker2    | DataNode, Stream Gateway                | Aktif  |

Semua node berada di dalam subnet `Subnet-Compute (10.0.1.0/24)` dan telah siap untuk proses instalasi Hadoop–Spark cluster.

---

#### 8. Catatan Penting

1. Untuk efisiensi biaya, aktifkan fitur **Auto-shutdown** pada setiap VM melalui portal Azure (misalnya pukul 23.00 WIB).
2. Gunakan label atau tag yang konsisten, seperti:

   ```
   project=insightera
   phase=sprint1
   env=dev
   ```
3. Simpan daftar IP dan kredensial SSH di repositori dokumentasi internal agar memudahkan konfigurasi otomatis pada tahap berikutnya.

---

Sprint 1 Bagian 2 ini menandai penyelesaian tahap pembuatan cluster virtual untuk lingkungan pengembangan INSIGHTERA. Seluruh node kini siap digunakan untuk instalasi Hadoop, Spark, dan konfigurasi jaringan antar node pada bagian selanjutnya.
Langkah berikutnya akan dilanjutkan pada **Sprint 1 Bagian 3: Instalasi dan Konfigurasi Hadoop 3.4.2 serta Spark 4.0.1 pada seluruh node cluster.**
