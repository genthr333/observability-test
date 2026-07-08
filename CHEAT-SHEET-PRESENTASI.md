# Cheat Sheet Presentasi (buat dipegang, bukan buat dibaca ke audiens)

## Kalimat pembuka per soal (hafalin garis besarnya, bukan kata per kata)

**Soal 1:** "Prinsip yang kami pakai shift-left — security dicek sedini mungkin di pipeline, sebelum masuk production, pakai fasilitas yang sudah ada: Git untuk source control & hook, Jenkins untuk otomasi tiap stage security, Kubernetes untuk deploy & runtime protection."

**Soal 2:** "Karena aplikasi banyak dan tersebar internal maupun internet-facing, kami butuh satu titik pusat untuk log-metric-trace, supaya developer tidak perlu exec ke pod satu-satu, dan tim bisa tahu ada masalah sebelum user komplain."

## Kemungkinan pertanyaan tim & jawaban singkat

**"Kenapa Trivy, bukan Snyk/lainnya?"**
→ Open-source, gratis, sekali tool bisa scan dependency DAN image, cocok untuk environment yang belum mau bayar SaaS.

**"Kalau severity threshold terlalu ketat, pipeline jadi lambat/sering fail dong?"**
→ Makanya di-set threshold Critical/High saja yang block, Medium/Low dicatat sebagai backlog technical debt, tidak menghentikan delivery.

**"SonarQube/Vault itu butuh infra tambahan, siapa yang maintain?"**
→ Semua bisa dijalankan sebagai workload di Kubernetes yang sudah ada (Helm chart tersedia untuk semuanya), tidak butuh server baru di luar cluster.

**"Bedanya SAST sama DAST apa lagi selain kapan dijalankan?"**
→ SAST tahu baris kode mana yang salah tapi rawan false positive; DAST hasil temuan lebih nyata (karena benar-benar menyerang app running) tapi tidak tahu baris kode. Makanya dipakai dua-duanya, saling melengkapi.

**"Kalau developer nge-bypass pre-commit hook di laptop gimana?"**
→ Makanya ada layer kedua di Jenkins (CI) dan idealnya layer ketiga di server Git (pre-receive hook) — jadi walau di-bypass di lokal, masih ketangkep di server.

**"ELK itu berat, resource-nya gimana?"**
→ Fluent Bit dipilih (bukan Logstash) karena ringan untuk DaemonSet; Elasticsearch retention di-set terbatas (misal 30 hari) supaya storage tidak membengkak terus.

**"Bagaimana cara tahu 'lambat'-nya itu di DB atau di aplikasi?"**
→ Ini justru kenapa perlu tracing (OpenTelemetry), karena itu breakdown waktu response per komponen — bukan cuma tebak-tebak dari gejala.

**"Ini semua butuh berapa lama implementasinya?"**
→ Jujur saja kalau ditanya: bertahap. Bisa mulai dari yang paling murah/cepat dulu (Gitleaks + Trivy di Jenkins, itu setup-nya cepat), baru masuk ke yang lebih besar (Vault, ELK, tracing) sebagai fase berikutnya.

## Kalau ditanya sesuatu yang lo belum yakin
Jangan sok tahu. Bilang jujur: "Itu belum pernah gua implementasikan langsung, tapi konsepnya begini... kalau di production nanti perlu divalidasi lagi." Ini lebih kredibel daripada ngarang.
