# Tutorial: Ingress Controller (Traefik)

**Prasyarat:** Selesaikan dulu [Tutorial: Memulai dengan K3s](000-k3s-getting-started.md). K3s sudah berjalan dan `kubectl` siap digunakan.

**Tujuan:** Memahami konsep Ingress controller dan memastikan Traefik bawaan K3s sudah berjalan.

---

## Analoginya sederhana

Ingress controller adalah komponen yang menerima trafik dari luar cluster dan meneruskannya ke Service yang tepat berdasarkan aturan Ingress.

Bayangkan:

- **Service** itu seperti pintu belakang tiap toko (hanya bisa diakses dari dalam kompleks).
- **Ingress controller** itu seperti satpam di gerbang kompleks yang mengarahkan tamu ke toko yang benar berdasarkan nama domain atau path.

Jadi, banyak aplikasi bisa diakses dari satu pintu masuk (satu IP) dengan rapi dan mudah diatur.

Di K3s, Ingress controller **sudah tersedia** secara default: **Traefik**. Karena itu, bagian selanjutnya fokus ke Traefik sebagai Ingress controller bawaan.

---

## Gambaran alur (schema sederhana)

```
Internet
   |
   v
DNS (app.example.com)
   |
   v
Ingress Controller (Traefik)
   |
   v
Ingress rules (host/path)
   |
   v
Service (ClusterIP)
   |
   v
Pod (aplikasi)
```

Domain/URL masuk ke Traefik, Traefik membaca aturan Ingress, lalu meneruskan ke Service yang tepat, dan Service mengarah ke Pod.

---

## 1. Cek Traefik sudah jalan

Cek statusnya dengan:

```bash
kubectl get pods -n kube-system | grep traefik
```

Jika status **Running**, Traefik siap dipakai.

---

## Ringkasan

Yang sudah kamu lakukan:

1. Memahami peran Ingress controller dengan analogi sederhana.
2. Memahami alur Ingress dari domain sampai Pod.
3. Mengecek Traefik bawaan K3s.

**Langkah selanjutnya:**

- **Tutorial membuat aplikasi:** [Membuat Aplikasi (Express + DB + Redis)](005-start-app-express.md) — siapkan aplikasi sebelum dipasang Ingress dan domain.
- **Tutorial domain & Ingress:** ikuti tutorial domain berikutnya untuk membuat Ingress aplikasi dan mengatur DNS.
- **Tutorial HTTPS:** [Let's Encrypt dengan cert-manager](003-letsencrypt-cert-manager.md) — menambahkan TLS pada Ingress.
- **Reference Traefik:** [Traefik (Ingress K3s)](../reference/001-traefik.md) — detail konfigurasi Traefik.
