# Tutorial: Helm

**Prasyarat:** Selesaikan dulu [Tutorial: Memulai dengan K3s](000-k3s-getting-started.md). Pastikan `kubectl` sudah bisa akses cluster dan `KUBECONFIG` sudah diarahkan.

**Tujuan:** Memasang Helm dan memastikan Helm siap dipakai untuk instalasi aplikasi Kubernetes.

---

## Kenapa pakai Helm?

Helm adalah package manager untuk Kubernetes. Helm memudahkan instalasi aplikasi lewat chart, jadi kamu tidak perlu mengelola banyak manifest YAML secara manual.

---

## Persiapan

Buat folder kerja agar rapi:

```bash
mkdir -p deploy-tools/helm
cd deploy-tools/helm
```

---

## 1. Install Helm (resmi)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

---

## 2. Cek versi

```bash
helm version
```

Contoh output:

```
version.BuildInfo{Version:"v3.x.x", ...}
```

---

## 3. Kunci permission binary (opsional, disarankan)

```bash
chmod +x /usr/local/bin/helm
```

---

## 4. Quick test

1. Cari repo (harusnya kosong dulu):

```bash
helm search repo bitnami/nginx
```

2. Tambah repo:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

3. Cari lagi:

```bash
helm search repo bitnami/nginx
```

Jika hasil muncul, Helm sudah siap dan aman dipakai.

4. Jika tidak ingin menyimpan repo test, hapus:

```bash
helm repo remove bitnami
```

---

## Ringkasan

Yang sudah kamu lakukan:

1. Menginstal Helm.
2. Memverifikasi Helm berjalan dengan benar.

**Langkah selanjutnya:**

- **Tutorial Longhorn:** [Longhorn (Persistent Storage)](002-longhorn.md) — menyiapkan persistent storage untuk aplikasi stateful.
- **How-to Helm:** (opsional) gunakan chart saat deploy aplikasi.
