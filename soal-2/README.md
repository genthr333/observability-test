# Soal 2 Observability (Log, Metric, Trace)

Kondisi: banyak aplikasi berjalan di Kubernetes, ada yang di jaringan internal, ada yang exposed ke internet.

## Daftar Isi

1. [Monitoring Aktif & Pasif Deteksi Aplikasi Down](#1-monitoring-aktif--pasif--deteksi-aplikasi-down)
2. [Pengelolaan Log ELK Stack](#2-pengelolaan-log--elk-stack)
3. [Mengukur Performa Database (Metric)](#3-mengukur-performa-database-metric)

---

## 1. Monitoring Aktif & Pasif Deteksi Aplikasi Down

Tujuan: tim tahu aplikasi down **secepat mungkin**, idealnya sebelum user komplain.

### Komponen inti
- **Prometheus** scrape metric dari semua aplikasi & node di cluster (jaringan internal).
- **kube-state-metrics** + **node-exporter** kondisi pod/node level (crashloop, resource, dsb).
- **Blackbox Exporter** probe endpoint dari luar (HTTP/TCP/ICMP check), penting untuk aplikasi yang **exposed ke internet**, karena ini simulasi pengecekan dari sisi user, bukan dari dalam cluster.
- **Alertmanager** routing alert ke Slack/Telegram/email/on-call (PagerDuty-style) berdasarkan severity.
- **Grafana** dashboard visual untuk semua metric di atas.

### Aktif vs Pasif

| | Internal (dalam cluster) | Internet-facing |
|---|---|---|
| **Aktif** (tim harus lihat sendiri) | Dashboard Grafana per aplikasi, dicek berkala | Uptime dashboard (Grafana + blackbox exporter results) |
| **Pasif** (sistem kasih tahu duluan) | Liveness/readiness probe Kubernetes otomatis restart pod bermasalah + Alertmanager kirim notifikasi kalau pod crashloop; Prometheus alert rule (misal error rate > threshold) | Blackbox exporter probe dari luar cluster tiap interval pendek (misal 30 detik) → alert kalau endpoint tidak respons / status code bukan 2xx |

### Kenapa dibedakan internal vs internet-facing
Aplikasi internal biasanya cukup dicek "dari dalam" (pod ke pod), tapi **aplikasi yang di-expose ke internet** perlu dicek juga "dari luar" karena bisa saja pod-nya sehat tapi ada masalah di layer lain (DNS, load balancer, firewall, NGINX reverse proxy/SSL) yang bikin user tidak bisa akses padahal dari sisi cluster semua terlihat normal.

---

## 2. Pengelolaan Log ELK Stack

### Masalah saat ini
Developer harus `kubectl exec`/`kubectl logs` masuk ke pod satu-satu untuk cari error. Ini tidak scalable kalau aplikasi banyak, pod-nya sering restart (log lama hilang begitu pod mati), dan susah korelasi error lintas service.

### Solusi: centralized logging (EFK/ELK)

```
Pod/Container (semua namespace)
        │  (stdout/stderr log)
Fluent Bit (DaemonSet — jalan di tiap node, "nempel" ambil log semua container)
        │
Elasticsearch (index & simpan log terpusat)
        │
Kibana (UI untuk search, filter, dashboard)
```

- **Fluent Bit** (lebih ringan dari Logstash) dipasang sebagai **DaemonSet** di Kubernetes otomatis ambil log dari semua pod tanpa developer perlu setup apa-apa per aplikasi.
- **Elasticsearch** simpan log terpusat dengan retention policy (misal simpan 30 hari, lalu archive/hapus) jadi log tidak hilang walau pod restart/dihapus.
- **Kibana** kasih UI: developer tinggal search `service:checkout AND level:error` misalnya, tidak perlu tahu pod mana / exec ke mana.


---

## 3. Mengukur Performa Database (Metric)

Case: ada laporan "aplikasi lambat", butuh cara ukur di sisi database.

### Solusi
- **Exporter khusus database** ke Prometheus:
  - `mysqld_exporter` untuk MySQL/MariaDB
  - `postgres_exporter` untuk PostgreSQL
- Metric yang dipantau:
  - **Query latency** (rata-rata & p95/p99 waktu eksekusi query)
  - **Slow query count** (query yang lebih lambat dari threshold, misal >1 detik)
  - **Active connections** vs **connection pool limit** (kalau mentok, request baru harus antri = terasa lambat)
  - **Lock wait time** (transaksi saling tunggu)
  - **Replication lag** (kalau ada read replica, data yang dibaca bisa "telat")
- **Grafana dashboard** khusus database untuk visualisasi semua metric di atas.
- **Slow query log** diaktifkan di database, dikirim ke ELK juga supaya bisa lihat **query spesifik** mana yang lambat, bukan cuma angka agregat.
