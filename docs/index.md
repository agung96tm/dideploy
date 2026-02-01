# Dideploy — K3s

Panduan K3s untuk jalankan Kubernetes di server sendiri (VPS atau bare metal) — dari pasang sampai storage, load balancer, dan ingress. Semua dalam Bahasa Indonesia.

## Kenapa ada situs ini?

Banyak yang mau jalankan Kubernetes di server sendiri (VPS atau bare metal) pakai K3s, tapi panduan sering tersebar dan kebanyakan bahasa Inggris. Situs ini ada untuk **ngehimpun panduan K3s** dalam satu tempat, terstruktur, dan dalam **Bahasa Indonesia** — dari pasang K3s, persistent storage (Longhorn), LoadBalancer (MetalLB), sampai Ingress (Traefik). Tujuannya: siapa saja bisa ikut langkah yang sama untuk deploy di lingkungan self-hosted tanpa harus nebak-nebak dokumentasi.

## Isi dokumentasi

- **Tutorials** — Mulai dari nol: pasang K3s, konfigurasi kubectl, deploy contoh.
- **How-to Guides** — Langkah praktis: cluster multi-node, MetalLB (LoadBalancer), dan lain-lain.
- **Reference** — Acuan singkat: Traefik (Ingress K3s), konfigurasi.
- **Explanation** — Penjelasan konsep dan arsitektur.

## Progress dokumentasi

Dokumentasi masih terus ditambah. Perkiraan progress saat ini:

**20%** — ████░░░░░░░░░░░░░░░░

- Sudah: K3s getting started, Longhorn (tutorial + how-to), MetalLB, Traefik & config (referensi), Architecture (explanation).
- Rencana: lebih banyak how-to, troubleshooting, contoh deploy app lengkap, dan penjelasan konsep.

---

Mulai dari [Memulai dengan K3s](tutorials/000-k3s-getting-started.md) kalau baru pertama kali pakai K3s.
