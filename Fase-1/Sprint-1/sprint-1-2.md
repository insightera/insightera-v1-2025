### Sprint 1 – Bagian 2

**Pembuatan Virtual Machine Cluster (Master, Management, Worker 1, dan Worker 2) – Ubuntu 24.04 LTS**

Bagian ini melanjutkan hasil konfigurasi jaringan dan resource group pada Sprint 1 Bagian 1. Tujuan dari bagian ini adalah membuat empat buah Virtual Machine (VM) yang akan berfungsi sebagai node utama dalam cluster hybrid INSIGHTERA. Keempat VM ini akan ditempatkan pada subnet `Subnet-Compute` di dalam VNet yang telah dibuat sebelumnya, dengan spesifikasi ringan untuk tahap pengembangan (developer cluster).

---

#### 1. Spesifikasi Virtual Machine

Semua VM menggunakan sistem operasi Ubuntu 24.04 LTS, dengan konfigurasi spesifikasi sebagai berikut:

| Node          | Peran                                        | vCPU | RAM  | Disk  | IP Privat |
| ------------- | -------------------------------------------- | ---- | ---- | ----- | --------- |
| VM-Master     | NameNode, ResourceManager, Spark Master      | 1    | 2 GB | 64 GB | 10.0.1.7  |
| VM-Management | Airflow, PostgreSQL, Hive Metastore, Grafana | 1    | 2 GB | 64 GB | 10.0.1.8  |
| VM-Worker1    | DataNode, NodeManager, Spark Worker          | 1    | 2 GB | 64 GB | 10.0.1.9  |
| VM-Worker2    | DataNode, NodeManager, Stream Gateway        | 1    | 1 GB | 64 GB | 10.0.1.10 |

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
az network nic create \
  --name NIC-Master \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --private-ip-address 10.0.1.7 \
  --network-security-group NSG-Insightera

az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Master \
  --image Ubuntu2404 \
  --size Standard_B1ms \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --nics NIC-Master \
  --os-disk-size-gb 64 \
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
  --public-ip-address VM-ManagementPublicIP \
  --public-ip-sku Standard \
  --tags project=insightera role=management env=dev
```

```bash
# VM-Worker1
az network nic create \
  --name NIC-Worker1 \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --private-ip-address 10.0.1.9 \
  --network-security-group NSG-Insightera

az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Worker1 \
  --image Ubuntu2404 \
  --size Standard_B1ms \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --nics NIC-Worker1 \
  --os-disk-size-gb 64 \
  --tags project=insightera role=worker env=dev
```

```bash
# VM-Worker2
az network nic create \
  --name NIC-Worker2 \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  --subnet Subnet-Compute \
  --private-ip-address 10.0.1.10 \
  --network-security-group NSG-Insightera

az vm create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VM-Worker2 \
  --image Ubuntu2404 \
  --size Standard_A1_v2 \
  --admin-username insightera \
  --ssh-key-values ~/.ssh/insightera.pub \
  --nics NIC-Worker2 \
  --os-disk-size-gb 64 \
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

Gunakan kunci SSH yang telah dibuat untuk mengakses VM Management:

```bash
ssh -i ~/.ssh/insightera insightera@<PublicIP_VM-Management>
```

Setelah berhasil masuk ke VM Master, uji koneksi antar-node:

```bash
ping -c 2 10.0.1.8
ping -c 2 10.0.1.9
ping -c 2 10.0.1.10
```

Jika semua node dapat saling berkomunikasi, jaringan internal cluster telah berfungsi dengan baik.
Jika tidak maka lakukan ini:

## **Langkah 1 — Dapatkan public key dari VM-Management**

1. Masuk ke **VM-Management** via SSH (dari laptop):

   ```bash
   ssh insightera@<Public-IP-Management>
   ```
2. Lihat isi public key:

   ```bash
   cat ~/.ssh/insightera.pub
   ```
3. **Salin seluruh output** (dimulai dari `ssh-rsa AAAA...` sampai akhir).


## **Langkah 2 — Buka VM-Master di Azure Portal**

1. Buka portal: [https://portal.azure.com](https://portal.azure.com)
2. Masuk ke **Resource Group → RG-Datalakehouse-Insightera → VM-Master**
3. Di menu kiri, pilih **Operations → Run Command**
4. Pilih **RunShellScript**


## **Langkah 3 — Tambahkan public key ke VM-Master**

Di jendela *Run Command Script*, tempelkan perintah ini:

```bash
mkdir -p /home/insightera/.ssh
chmod 700 /home/insightera/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDqC9E... insightera@VM-Management" >> /home/insightera/.ssh/authorized_keys
chmod 600 /home/insightera/.ssh/authorized_keys
chown -R insightera:insightera /home/insightera/.ssh
```

> **Ganti** isi `"ssh-rsa AAAA... insightera@VM-Management"` dengan hasil asli dari langkah 1.

Klik **Run** 


## **Langkah 4 — Uji koneksi dari VM-Management**

Kembali ke terminal di VM-Management:

```bash
ssh -i ~/.ssh/insightera insightera@10.0.1.7
```

Sekarang seharusnya **langsung berhasil login ke VM-Master tanpa error `Permission denied`** 🎉


## **Ulangi untuk node lain (Worker1, Worker2)**

Kamu bisa ulangi langkah **2–4** untuk:

* `VM-Worker1`
* `VM-Worker2`

Cukup ganti target VM di Azure Portal dan jalankan perintah yang sama.


## Tips Tambahan (jika sering perlu)

Kalau sering ingin mengubah authorized key antar-node, aktifkan fitur:
**VM → Reset password → SSH public key**

* Masuk ke: VM-Master → *Support + Troubleshooting → Reset password*
* Mode: `SSH public key`
* Username: `insightera`
* Paste isi key `.pub` dari Management

Azure akan otomatis menulis ulang file `authorized_keys` di VM-Master.

Selanjutnya buat lagi insightera.pub di VM Master:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/insightera -N ""
```

lanjutkan dengan melihat isi

   ```bash
   cat ~/.ssh/insightera.pub
   ```
**Salin seluruh output** (dimulai dari `ssh-rsa AAAA...` sampai akhir).

Masuk ke portal azure dan run seperti code pada langkah 3
```bash
mkdir -p /home/insightera/.ssh
chmod 700 /home/insightera/.ssh
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDqC9E... insightera@VM-Management" >> /home/insightera/.ssh/authorized_keys
chmod 600 /home/insightera/.ssh/authorized_keys
chown -R insightera:insightera /home/insightera/.ssh
```
hanya saja ssh-rsa nya ganti yang di buat dari VM Master, run ini di VM-Worker1 dan VM-Worker2

lalu coba masuk dari master ke worker dengan:
- Worker 1
```bash
ssh -i ~/.ssh/insightera insightera@10.0.1.9
```
lalu ketik exit untuk keluar dari worker1 ke master, lalu periksa lagi:

- Worker 2
```bash
ssh -i ~/.ssh/insightera insightera@10.0.1.10
```

lalu exit kembali ke master dan management


#### 6. Menambahkan Host Mapping

Agar setiap node dapat saling mengenali tanpa IP, tambahkan entri ke dalam file `/etc/hosts` pada masing-masing VM (Master dan Management):

```
10.0.1.7 master
10.0.1.9 worker1
10.0.1.10 worker2
```

Gunakan perintah berikut di VM Master untuk mengedit:

```bash
sudo nano /etc/hosts
```

Simpan perubahan dengan `Ctrl + O` lalu keluar dengan `Ctrl + X`.

lanjut agar bisa langsung memanggil denganb `ssh master` atau maka edit konfigurasi ini di 

```bash
nano ~/.ssh/config
```
lanjutkan dengan paste ini:

```bash
# Konfigurasi SSH untuk cluster INSIGHTERA

Host master
    HostName 10.0.1.7
    User insightera
    IdentityFile ~/.ssh/insightera

Host worker1
    HostName 10.0.1.9
    User insightera
    IdentityFile ~/.ssh/insightera

Host worker2
    HostName 10.0.1.10
    User insightera
    IdentityFile ~/.ssh/insightera
```

lalu keluar dan simpan.

lakukan hal yang sama seperti di atas di VM Master. lalu coba `ssh worker1` atau `ssh worker2` maka akan masuk ke VM Worker 1 atau Woker 2.

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
