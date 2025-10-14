# Desain Pelaksanaan Agile Scrum - Proyek INSIGHTERA

## **FASE I: Infrastruktur dan Data Lakehouse (Hybrid Core)**

| Sprint | Periode | Durasi | Fokus Pengembangan | Sprint Review | Sprint Retrospective | Output/Deliverable |
|--------|---------|--------|-------------------|---------------|---------------------|-------------------|
| **Sprint 1** | 1-10 Agustus 2025 | 10 hari | Penyiapan kluster VM, SSH, Hadoop, dan Spark pada Azure IaaS | 10 Agustus, 14:00-15:30 | 10 Agustus, 15:30-16:30 | Infrastruktur VM dan kluster Hadoop-Spark operational |
| **Sprint 2** | 11-17 Agustus 2025 | 7 hari | Integrasi HDFS dan Azure Data Lake (HybridMount) | 17 Agustus, 14:00-15:30 | 17 Agustus, 15:30-16:30 | Hybrid storage layer terintegrasi |
| **Sprint 3** | 18-24 Agustus 2025 | 7 hari | Setup Hive dan PostgreSQL Metastore, koneksi Hadoop-Spark | 24 Agustus, 14:00-15:30 | 24 Agustus, 15:30-16:30 | Metastore dan query engine siap pakai |
| **Sprint 4** | 25-31 Agustus 2025 | 7 hari | Instalasi Apache Airflow dan pembuatan pipeline DAG ingestion | 31 Agustus, 14:00-15:30 | 31 Agustus, 15:30-16:30 | Orchestration pipeline operational |
| **Sprint 5** | 1-7 September 2025 | 7 hari | Integrasi Synapse Analytics dan Power BI untuk visualisasi awal | 7 September, 14:00-15:30 | 7 September, 15:30-16:30 | Dashboard awal dan koneksi Synapse aktif |

---

## **FASE II: Federasi Data Mart (15 Unit Kerja ITERA)**

| Sprint | Periode | Durasi | Fokus Pengembangan | Sprint Review | Sprint Retrospective | Output/Deliverable |
|--------|---------|--------|-------------------|---------------|---------------------|-------------------|
| **Sprint 6** | 1-7 September 2025 | 7 hari | Pembuatan Template Schema dan Metadata Catalog (paralel Sprint 5) | 7 September, 16:00-17:30 | 7 September, 17:30-18:30 | Template schema standar dan metadata catalog |
| **Sprint 7** | 8-14 September 2025 | 7 hari | Data Mart Prioritas: Akademik, Keuangan, LPPM, LPMPP | 14 September, 14:00-15:30 | 14 September, 15:30-16:30 | 4 Data Mart prioritas operational |
| **Sprint 8** | 15-21 September 2025 | 7 hari | Data Mart: SDM, Kemahasiswaan, Keamanan Siber | 21 September, 14:00-15:30 | 21 September, 15:30-16:30 | 3 Data Mart tambahan terintegrasi |
| **Sprint 9** | 22-28 September 2025 | 7 hari | Data Mart Operasional: Infrastruktur, K3L, Kebun Raya, Fakultas | 28 September, 14:00-15:30 | 28 September, 15:30-16:30 | Data Mart operasional unit kerja |
| **Sprint 10** | 29 Sept-6 Okt 2025 | 8 hari | Harmonisasi dan Federasi Antar Data Mart (EDW Federation Layer) | 6 Oktober, 14:00-15:30 | 6 Oktober, 15:30-16:30 | Federasi 15 data mart dengan metadata bersama |

---

## **FASE III: Enterprise Data Warehouse (EDW) dan Business Intelligence Layer**

| Sprint | Periode | Durasi | Fokus Pengembangan | Sprint Review | Sprint Retrospective | Output/Deliverable |
|--------|---------|--------|-------------------|---------------|---------------------|-------------------|
| **Sprint 11** | 7-13 Oktober 2025 | 7 hari | Desain Model Dimensional Kimball untuk EDW | 13 Oktober, 14:00-15:30 | 13 Oktober, 15:30-16:30 | Model dimensional EDW (fact & dimension tables) |
| **Sprint 12** | 14-20 Oktober 2025 | 7 hari | Migrasi Data Federasi ke Azure Synapse EDW | 20 Oktober, 14:00-15:30 | 20 Oktober, 15:30-16:30 | EDW Synapse dengan data terintegrasi |
| **Sprint 13** | 21-27 Oktober 2025 | 7 hari | Pembuatan Model Analitik dan Dashboard Power BI (iterasi awal) | 27 Oktober, 14:00-15:30 | 27 Oktober, 15:30-16:30 | Dashboard BI interaktif untuk stakeholder |
| **Sprint 14** | 28 Okt-3 Nov 2025 | 7 hari | Integrasi Data Governance dengan Azure Purview | 3 November, 14:00-15:30 | 3 November, 15:30-16:30 | Data governance framework operational |
| **Sprint 15** | 4-10 November 2025 | 7 hari | Pengujian Kinerja dan Optimasi EDW (paralel Sprint 16) | 10 November, 14:00-15:30 | 10 November, 15:30-16:30 | EDW teroptimasi dengan performa tinggi |

---

## **FASE IV: AI dan Advanced Analytics**

| Sprint | Periode | Durasi | Fokus Pengembangan | Sprint Review | Sprint Retrospective | Output/Deliverable |
|--------|---------|--------|-------------------|---------------|---------------------|-------------------|
| **Sprint 16** | 4-10 November 2025 | 7 hari | Pengembangan Machine Learning Pipeline untuk Predictive Analytics | 10 November, 16:00-17:30 | 10 November, 17:30-18:30 | ML pipeline untuk prediksi |
| **Sprint 17** | 11-17 November 2025 | 7 hari | Implementasi Computer Vision Stream Analytics (K3L Domain) | 17 November, 14:00-15:30 | 17 November, 15:30-16:30 | Sistem computer vision K3L aktif |
| **Sprint 18** | 18-24 November 2025 | 7 hari | Pengembangan Text Mining dan NLP Dashboard (LPMPP Domain) | 24 November, 14:00-15:30 | 24 November, 15:30-16:30 | NLP dashboard untuk analisis teks |
| **Sprint 19** | 25-30 November 2025 | 6 hari | Peningkatan Data Governance dan Auto Lineage Tracking | 30 November, 14:00-15:30 | 30 November, 15:30-16:30 | Auto lineage tracking terimplementasi |
| **Sprint 20** | 25-30 November 2025 | 6 hari | Optimasi Sistem, Evaluasi Kinerja, Persiapan Scaling 2026 (paralel Sprint 19) | 30 November, 16:00-17:30 | 30 November, 17:30-18:30 | Sistem INSIGHTERA siap produksi |

---

## **Struktur Kegiatan Sprint (Daily Cycle)**

| Kegiatan | Waktu | Durasi | Peserta | Tujuan |
|----------|-------|--------|---------|---------|
| **Daily Scrum** | Setiap hari, 09:00 | 15 menit | Development Team, Scrum Master | Sinkronisasi progress, identifikasi hambatan |
| **Sprint Planning** | Hari pertama sprint, 09:00 | 2-4 jam | Product Owner, Scrum Master, Dev Team | Menentukan Sprint Goal dan Sprint Backlog |
| **Sprint Review** | Hari terakhir sprint, 14:00 | 1.5 jam | Seluruh Scrum Team + Stakeholder | Demo increment, terima feedback |
| **Sprint Retrospective** | Hari terakhir sprint, setelah Review | 1 jam | Scrum Master, Development Team | Evaluasi proses, identifikasi improvement |

---

## **Catatan Penting:**

1. **Sprint Paralel**: Sprint 5 & 6 serta Sprint 15 & 16 dan Sprint 19 & 20 berjalan paralel dengan tim berbeda
2. **Durasi Sprint Review**: 1.5 jam untuk presentasi hasil dan feedback stakeholder
3. **Durasi Sprint Retrospective**: 1 jam untuk refleksi internal tim
4. **Jadwal disesuaikan** dengan ketersediaan stakeholder kunci dari 15 unit kerja ITERA
5. **Definition of Done** harus disepakati di awal setiap sprint untuk memastikan kualitas deliverable

**Total Sprint**: 20 sprint dalam 4 bulan (Agustus-November 2025) dengan metodologi Agile Scrum yang iteratif dan adaptif.