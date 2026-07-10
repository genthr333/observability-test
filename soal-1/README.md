# Soal 1 DevSecOps

> Kondisi: fasilitas yang tersedia **Git, Jenkins, Kubernetes**.

## Daftar Isi

1. [Usulan Solusi DevSecOps](#1-usulan-solusi-devsecops)
2. [Static Test Cek Keamanan Code Sebelum Production (SAST)](#2-static-test--sast)
3. [Cek Credential yang Kecommit ke Code (Secret Scanning)](#3-cek-credential-di-code-secret-scanning)
4. [Cek Kerentanan Library / Dependency (Patch Management)](#4-cek-kerentanan-library-patch-management)
5. [Monitoring Aktif & Pasif untuk Pegawai](#5-monitoring-aktif--pasif)
6. [Manajemen Secret & Config](#6-manajemen-secret--config)

---

## 1. Usulan Solusi DevSecOps

Prinsip DevSecOps yang dipakai: **"shift-left"** security dicek sedini mungkin (di kode, bukan di production), dan **security jadi stage otomatis di pipeline**, bukan proses manual terpisah.

Dengan Git + Jenkins + Kubernetes, pipeline yang diusulkan:

```
1. Developer push code ke Git
        │
2. Pull Request ke branch main/develop
        │   Jenkins trigger otomatis (webhook)
3. STAGE: Secret Scanning       = Gitleaks/TruffleHog (cek credential bocor)
        |
4. STAGE: SAST                  = SonarQube / Semgrep (cek bug & vuln di source code)
        |
5. STAGE: SCA (Dependency Scan) = Trivy / OWASP Dependency-Check (cek library rentan)
        |
6. Build Docker Image
        |
7. STAGE: Image Scanning        = Trivy image scan (cek CVE di base image & layer)
        |
8. Push image ke Container Registry (private)
        |
9. Deploy ke Kubernetes namespace STAGING (via Helm)
        |
10. STAGE: DAST                 = OWASP ZAP baseline scan ke staging URL
        |
11. Manual approval / gate → Deploy ke Kubernetes PRODUCTION
        |
12. Observability: Prometheus + Grafana + Falco (runtime monitoring) mengawasi terus
```

**Kenapa strukturnya begini:**
- Stage 3–5 jalan **sebelum build image** kalau ada credential bocor atau vuln kritis, pipeline **fail cepat**, tidak buang waktu build.
- Stage 7 (image scan) penting karena base image (misal `node:18`, `python:3.11`) juga bisa punya CVE, bukan cuma kode kita.
- Stage 10 (DAST) baru bisa jalan setelah app benar-benar running di staging, karena DAST butuh menyerang aplikasi yang hidup.
- Ada **gate/approval manual** sebelum ke production sebagai safety net terakhir Jenkins bisa pakai `input` step untuk ini.
- Semua stage security **tidak block selamanya**: pipeline dikonfigurasi dengan **severity threshold** (misal: fail kalau ada CVE Critical/High, warning-only kalau Medium/Low) supaya tidak menghambat kecepatan delivery secara berlebihan.

---

## 2. Static Test SAST

**SAST (Static Application Security Testing)** = analisis source code **tanpa menjalankan aplikasinya**, dilakukan saat development/CI, sebelum masuk production.

### Tools yang diusulkan
| Tool | Kelebihan | Catatan |
|---|---|---|
| **SonarQube** | Dashboard lengkap, quality gate, support banyak bahasa, ada Jenkins plugin resmi | Butuh server sendiri (bisa dijalankan di Kubernetes yang sudah ada) |
| **Semgrep** | Ringan, cepat, rule custom mudah, community rules banyak | Cocok untuk quick-check di setiap PR |

### Implementasi di pipeline
- Dijalankan **setiap ada Pull Request / push ke branch utama**, sebelum stage build.
- Hasil scan dikirim sebagai **Quality Gate**: jika ada vulnerability Critical/High, Jenkins build **fail otomatis** dan PR tidak bisa di-merge.
- Report bisa diintegrasikan ke PR sebagai comment otomatis (developer langsung tahu baris mana yang bermasalah).

### Yang bisa ditemukan SAST
- SQL Injection di source code
- Hardcoded credential/API key
- Insecure function usage (misal `eval()`, deserialization tidak aman)
- Dependency dengan known vulnerability (overlap dengan SCA)

---

## 3. Secret Scanning (Cek Credential di Code)

Masalah : developer sering tidak sengaja commit `.env`, API key, password DB, private key ke Git.

### Defense in depth

**Layer 1 Pre-commit (di laptop developer, sebelum sempat push)**
- Install **Gitleaks** atau **detect-secrets** sebagai git pre-commit hook.
- Kalau ada pattern yang match credential (API key, AWS secret, private key, dsb), commit **ditolak di lokal** belum sempat ke server sama sekali.

**Layer 2 CI Pipeline (Jenkins)**
- Stage awal Jenkins pipeline menjalankan **Gitleaks scan** ke seluruh history branch yang dipush.
- Kalau ketemu secret build fail, notifikasi ke channel (Slack/Telegram) + PR di-block.

**Layer 3 Server side / Git hosting**
- Kalau pakai GitHub/GitLab: aktifkan **push protection** / secret scanning bawaan.
- Kalau self-hosted Git (Gitea/GitLab CE): pasang **pre-receive hook** di server yang menjalankan gitleaks juga, supaya tidak bisa dibypass dengan skip pre-commit hook di lokal.

### Kalau sudah terlanjur kecommit
- Rotate/ganti credential yang bocor **segera** (anggap sudah kompromi, walau history di-rewrite).
- `git filter-repo` / BFG Repo-Cleaner untuk bersihkan history.

---

## 4. Patch Management

Ini disebut **SCA (Software Composition Analysis)**: cek semua dependency/library pihak ketiga yang dipakai aplikasi, dicocokkan dengan database CVE.

### Tools
| Tool | Cakupan |
|---|---|
| **Trivy** | Scan dependency file (package.json, go.mod, requirements.txt) **DAN** Docker image sekaligus paling praktis untuk stack Git+Jenkins+K8s |
| **OWASP Dependency Check** | Fokus dependency scan, laporan detail per-CVE |
| **Snyk** | SaaS, dashboard bagus, ada free tier |

### Alur patch management
1. Stage Jenkins jalankan `trivy fs .` (scan source) dan `trivy image <image>` (scan image) setiap pipeline jalan.
2. Severity threshold: **Critical/High = block pipeline**, Medium/Low = catat sebagai backlog.
3. **Scheduled scan mingguan** (bukan cuma pas ada push) karena CVE baru bisa muncul kapan saja untuk dependency yang sama, walau kodenya tidak berubah. Ini pakai Jenkins cron job terpisah.
4. Untuk update dependency otomatis: pakai **Renovate Bot** atau **Dependabot** (kalau self hosted, Renovate bisa dijalankan sebagai job) bikin PR otomatis kalau ada versi patch tersedia, developer tinggal review & merge.
5. Base image di Dockerfile dipin ke versi spesifik dan di-rebuild rutin (bukan pakai `latest`), supaya patch OS-level (misal Debian/Alpine) juga ikut ter-update.

---

## 5. Monitoring Aktif & Pasif

**Aktif** = pegawai/tim harus proaktif mengecek (dashboard, report berkala).
**Pasif** = sistem yang memberi tahu pegawai tanpa diminta (notifikasi/alert).

| Mode | Implementasi |
|---|---|
| **Aktif** | Dashboard Grafana (security posture: jumlah vuln open, trend per severity), report mingguan dari Trivy/SonarQube diexport ke Confluence/Slack channel, security scorecard per aplikasi |
| **Pasif** | Alertmanager/Jenkins notification ke Slack/Telegram/email saat pipeline fail karena security issue; Falco (runtime security di Kubernetes) kirim alert real-time kalau ada aktivitas mencurigakan di container (misal shell dibuka di pod production); Kubernetes audit log dikirim ke SIEM/ELK untuk deteksi anomali akses |

---

## 6. Manajemen Secret & Config

Prinsip: **secret tidak boleh pernah ada di Git**, dan **config dipisah dari code**.

### Config
Config (non-sensitif: URL, feature flag) taruh di Kubernetes ConfigMap, beda environment (dev/staging/prod) pakai file values beda di Helm.

### Secret
Secret (sensitif: password, API key) jangan pernah ditulis di kode atau dicommit ke Git. Taruh di Kubernetes Secret, atau kalau mau lebih aman pakai Vault. Nilai secret di-inject ke aplikasi saat runtime (lewat environment variable atau file), bukan ditulis manual di source code. lalu kredensial yang dipakai Jenkins sendiri (misal password ke registry, kubeconfig) disimpan di Jenkins Credentials Store, bukan ditulis langsung di Jenkinsfile
