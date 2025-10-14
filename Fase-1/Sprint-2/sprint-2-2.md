### Sprint 2 – Bagian 2

**Pembuatan Service Principal dan Konfigurasi Autentikasi OAuth2 untuk Hadoop–ADLS**

Bagian ini melanjutkan proses integrasi Hybrid Data Lakehouse INSIGHTERA dengan menyiapkan **mekanisme autentikasi aman** antara Hadoop–Spark Cluster dan Azure Data Lake Storage Gen2 (ADLS Gen2).
Autentikasi ini menggunakan **Service Principal** yang terdaftar di **Azure Active Directory (Azure AD)** dan metode **OAuth2 Client Credential Flow**, sehingga sistem dapat mengakses penyimpanan cloud tanpa perlu memasukkan kredensial secara manual.

---

#### 1. Tujuan Bagian Ini

1. Membuat Service Principal khusus INSIGHTERA (`sp-insightera`) di Azure AD.
2. Memberikan hak akses yang sesuai pada Storage Account ADLS (`insighteradata`).
3. Mendapatkan kredensial OAuth2 yang terdiri dari `client_id`, `tenant_id`, dan `client_secret`.
4. Menguji konektivitas menggunakan kredensial tersebut melalui Azure CLI dan Hadoop command.
5. Menyiapkan file konfigurasi dasar untuk digunakan pada Sprint 2 Bagian 3 (Hybrid Mount).

---

#### 2. Pembuatan Service Principal di Azure AD

Jalankan perintah berikut untuk membuat Service Principal baru:

```bash
az ad sp create-for-rbac \
  --name sp-insightera \
  --role "Storage Blob Data Contributor" \
  --scopes /subscriptions/<SUBSCRIPTION_ID>/resourceGroups/RG-Datalakehouse-Insightera \
  --sdk-auth
```

Output yang dihasilkan akan berbentuk JSON, berisi kredensial berikut:

```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy",
  "subscriptionId": "zzzzzzzz-zzzz-zzzz-zzzz-zzzzzzzzzzzz",
  "tenantId": "aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
}
```

Simpan informasi ini secara aman ke dalam file konfigurasi di VM Management:

```
/opt/insightera/config/azure/service_principal.json
```

---

#### 3. Penjelasan Parameter

| Parameter          | Keterangan                                                |
| ------------------ | --------------------------------------------------------- |
| **clientId**       | ID aplikasi yang digunakan Hadoop untuk autentikasi       |
| **clientSecret**   | Kata sandi aplikasi (harus dirahasiakan)                  |
| **tenantId**       | ID direktori tenant Azure                                 |
| **subscriptionId** | ID langganan Azure tempat sumber daya INSIGHTERA berada   |
| **role**           | Storage Blob Data Contributor (akses baca dan tulis blob) |

---

#### 4. Verifikasi Role Assignment

Untuk memastikan Service Principal telah diberikan hak akses dengan benar:

```bash
az role assignment list \
  --assignee sp-insightera \
  --all -o table
```

Pastikan hasilnya menampilkan peran **Storage Blob Data Contributor** dengan scope ke resource group `RG-Datalakehouse-Insightera` atau langsung ke storage account `insighteradata`.

---

#### 5. Uji Token Autentikasi OAuth2

Gunakan kredensial Service Principal untuk mendapatkan token OAuth2:

```bash
curl -X POST \
  -d "grant_type=client_credentials&client_id=<clientId>&client_secret=<clientSecret>&resource=https://storage.azure.com/" \
  https://login.microsoftonline.com/<tenantId>/oauth2/token
```

Jika berhasil, akan muncul output JSON yang memuat `access_token` seperti berikut:

```json
{
  "token_type": "Bearer",
  "expires_in": "3599",
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIs..."
}
```

Token ini akan digunakan Hadoop untuk otorisasi otomatis terhadap ADLS.

---

#### 6. Pengujian Akses Service Principal ke ADLS

Gunakan kredensial yang sama untuk menguji akses ADLS Gen2 menggunakan Azure CLI:

```bash
az storage fs list \
  --account-name insighteradata \
  --auth-mode login \
  --query [].name
```

Jika daftar container (`bronze`, `silver`, `gold`) muncul, maka autentikasi berhasil.

---

#### 7. Menyiapkan File Konfigurasi Hadoop untuk Autentikasi OAuth2

Buat atau edit file konfigurasi di VM Master:
`/opt/hadoop/etc/hadoop/core-site.xml`

Tambahkan parameter berikut:

```xml
<configuration>
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
</configuration>
```

Simpan dan salin file ini ke semua node cluster (Master, Worker1, Worker2):

```bash
scp /opt/hadoop/etc/hadoop/core-site.xml worker1:/opt/hadoop/etc/hadoop/
scp /opt/hadoop/etc/hadoop/core-site.xml worker2:/opt/hadoop/etc/hadoop/
```

---

#### 8. Pengujian Integrasi Dasar Hadoop–ADLS

Uji konektivitas dari Hadoop ke ADLS:

```bash
hdfs dfs -ls abfs://bronze@insighteradata.dfs.core.windows.net/
```

Jika konfigurasi benar, hasilnya akan menampilkan daftar folder yang sudah dibuat sebelumnya pada Sprint 2 Bagian 1.

Lakukan juga uji tulis file:

```bash
echo "hybrid test file" > test.txt
hdfs dfs -put test.txt abfs://bronze@insighteradata.dfs.core.windows.net/akademik/
```

Verifikasi di Azure Portal atau Storage Explorer untuk memastikan file `test.txt` muncul di container `bronze/akademik`.

---

#### 9. Keamanan dan Penyimpanan Kredensial

* Jangan menyimpan nilai `clientSecret` dalam repositori publik.

* Simpan file konfigurasi di direktori:

  ```
  /opt/insightera/config/azure/
  ```

  dengan permission terbatas:

  ```bash
  chmod 600 service_principal.json
  chmod 600 /opt/hadoop/etc/hadoop/core-site.xml
  ```

* Jika kredensial perlu diperbarui, gunakan perintah:

  ```bash
  az ad sp credential reset --name sp-insightera
  ```

---

#### 10. Hasil Akhir Sprint 2 Bagian 2

| Komponen           | Status        | Keterangan                                 |
| ------------------ | ------------- | ------------------------------------------ |
| Service Principal  | Dibuat        | sp-insightera aktif di Azure AD            |
| Role Assignment    | Berhasil      | Storage Blob Data Contributor              |
| Autentikasi OAuth2 | Valid         | Token diterbitkan dan diterima oleh Hadoop |
| File core-site.xml | Dikonfigurasi | Tersimpan di semua node cluster            |
| Tes Integrasi      | Sukses        | Hadoop dapat membaca & menulis ke ADLS     |

---

#### 11. Kesimpulan

Tahapan ini berhasil menyiapkan **mekanisme autentikasi aman** antara Hadoop dan Azure Data Lake menggunakan OAuth2 Client Credential Flow.
Cluster INSIGHTERA kini memiliki kredensial Service Principal khusus yang memungkinkan komunikasi langsung dengan ADLS tanpa interaksi manual pengguna.

Langkah berikutnya akan dilanjutkan pada **Sprint 2 – Bagian 3: Konfigurasi Hadoop untuk Akses ADLS dan Implementasi Hybrid Mount (abfs://)**, di mana integrasi penuh antara HDFS lokal dan ADLS cloud akan dikonfigurasi dan diuji sinkronisasi dua arah.
