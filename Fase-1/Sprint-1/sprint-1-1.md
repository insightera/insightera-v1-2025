# Sprint 1 – Bagian 1  
## Persiapan Lingkungan Azure CLI dan Konfigurasi Dasar Infrastruktur (macOS / Ubuntu)

Tahapan awal membangun infrastruktur hybrid INSIGHTERA dimulai dengan instalasi dan konfigurasi Azure CLI, autentikasi akun, pemilihan subscription, serta pembuatan **Resource Group**, **Virtual Network (VNet)**, dan empat **subnet utama**: Compute, Storage, Management, dan Bastion.

---

## Prasyarat

- macOS atau Ubuntu Linux dengan akses terminal  
- Akun Microsoft Azure aktif dengan role `Contributor`  
- Koneksi internet stabil  
- Hak akses SSH lokal  

---

## 1. Instalasi Azure CLI

### macOS
```bash
brew update
brew install azure-cli
````

Verifikasi:

```bash
az version
```

### Ubuntu

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

Verifikasi:

```bash
az version
```

---

## 2. Login ke Akun Azure

```bash
az login
```

Gunakan browser untuk autentikasi.
Jika di server tanpa GUI:

```bash
az login --use-device-code
```

---

## 3. Memilih Subscription

```bash
az account list -o table
az account set --subscription "<Nama atau ID Subscription>"
az account show -o table
```

---

## 4. Membuat Resource Group

```bash
az group create \
  --name RG-Datalakehouse-Insightera \
  --location southeastasia
```

Cek hasil:

```bash
az group list -o table
```

---

## 5. Membuat Virtual Network dan Subnet INSIGHTERA

### 5.1 Membuat VNet utama

```bash
az network vnet create \
  --resource-group RG-Datalakehouse-Insightera \
  --name VNet-Datalakehouse-Insightera \
  --address-prefix 10.0.0.0/16 \
  --subnet-name Subnet-Compute \
  --subnet-prefix 10.0.1.0/24
```

### 5.2 Menambahkan Subnet Storage

```bash
az network vnet subnet create \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  --name Subnet-Storage \
  --address-prefix 10.0.2.0/24
```

### 5.3 Menambahkan Subnet Management

```bash
az network vnet subnet create \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  --name Subnet-Management \
  --address-prefix 10.0.3.0/24
```

### 5.4 Menambahkan Subnet Bastion

```bash
az network vnet subnet create \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  --name Subnet-Bastion \
  --address-prefix 10.0.4.0/24
```

Verifikasi seluruh subnet:

```bash
az network vnet subnet list \
  --resource-group RG-Datalakehouse-Insightera \
  --vnet-name VNet-Datalakehouse-Insightera \
  -o table
```

---

## 6. Membuat Network Security Group (NSG)

```bash
az network nsg create \
  --resource-group RG-Datalakehouse-Insightera \
  --name NSG-Insightera
```

Tambahkan aturan keamanan dasar:

```bash
az network nsg rule create \
  --resource-group RG-Datalakehouse-Insightera \
  --nsg-name NSG-Insightera \
  --name AllowSSH \
  --protocol Tcp --priority 1000 \
  --destination-port-ranges 22 \
  --access Allow

az network nsg rule create \
  --resource-group RG-Datalakehouse-Insightera \
  --nsg-name NSG-Insightera \
  --name AllowWebUI \
  --protocol Tcp --priority 1010 \
  --destination-port-ranges 8088 9870 4040 8080 \
  --access Allow
```

---

## 7. Hasil Akhir Sprint 1 Bagian 1

| Komponen              | Nama                          | Rentang IP                                | Keterangan                  |
| --------------------- | ----------------------------- | ----------------------------------------- | --------------------------- |
| **Resource Group**    | RG-Datalakehouse-Insightera   | –                                         | Kontainer semua sumber daya |
| **VNet**              | VNet-Datalakehouse-Insightera | 10.0.0.0/16                               | Jaringan utama cluster      |
| **Subnet-Compute**    | 10.0.1.0/24                   | Node Master & Worker                      |                             |
| **Subnet-Storage**    | 10.0.2.0/24                   | Integrasi ADLS / HDFS Gateway             |                             |
| **Subnet-Management** | 10.0.3.0/24                   | Airflow, Grafana, Metastore               |                             |
| **Subnet-Bastion**    | 10.0.4.0/24                   | Akses administratif aman                  |                             |
| **NSG-Insightera**    | –                             | Port 22, 8088, 9870, 4040, 8080 diizinkan |                             |

---

## 8. Troubleshooting

| Masalah                   | Penyebab                   | Solusi                                   |
| ------------------------- | -------------------------- | ---------------------------------------- |
| `az: command not found`   | Azure CLI belum diinstal   | Instal ulang CLI sesuai OS               |
| `Address prefix conflict` | Range IP sudah digunakan   | Ganti dengan blok lain, mis. 10.1.0.0/16 |
| `AuthorizationFailed`     | Izin akun kurang           | Pastikan role Contributor aktif          |
| `Resource already exists` | Nama RG/VNet sudah dipakai | Tambahkan suffix mis. `-Dev`             |

---

## Tahap Berikutnya

Lanjutkan ke **Sprint 1 – Bagian 2:**
📘 *Pembuatan VM Cluster (Master, Management, Worker 1–2) Ubuntu 24.04 LTS dengan spesifikasi 2 vCPU / 4 GB / 64 GB.*
