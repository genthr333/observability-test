# DevSecOps & Observability — Solution Proposal

> Jawaban technical test: usulan solusi DevSecOps dan Observability dengan fasilitas yang tersedia (**Git, Jenkins, Kubernetes**).

## Daftar Isi

1. [Soal 1 — DevSecOps](./soal-1-devsecops/README.md)
2. [Soal 2 — Observability (Log, Metric, Trace)](./soal-2-observability/README.md)

## Ringkasan Pendekatan

Kedua soal ini sebenarnya saling terkait: DevSecOps memastikan **apa yang masuk ke production itu aman**, Observability memastikan **kita tahu kondisi production setelah dia jalan**. Solusi di sini dibangun di atas 3 fasilitas yang sudah ada — tidak menambah tools yang butuh vendor baru/proses procurement, semuanya open-source dan bisa jalan di atas Kubernetes yang sudah ada.

```
        DEVSECOPS (sebelum production)              OBSERVABILITY (setelah production)
Git → Jenkins Pipeline → Kubernetes  ────────────▶   Kubernetes (running apps) → Monitoring/Log/Trace
   (cegah masalah masuk)                                (tahu kalau ada masalah)
```

Referensi tambahan soal 1 poin static test: [EkoEdyP/SAST-DAST](https://github.com/EkoEdyP/SAST-DAST)
