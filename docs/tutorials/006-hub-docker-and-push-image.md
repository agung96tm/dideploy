# Tutorial: Push Image ke Docker Hub

**Prasyarat:** Selesaikan dulu [Tutorial: Membuat Aplikasi (Express + DB + Redis)](005-start-app-express.md). Image aplikasi sudah berhasil di-build secara lokal.

**Tujuan:** Tag image dengan versi, login ke Docker Hub, lalu push agar bisa dipakai saat deploy di Kubernetes.

---

## Kenapa harus push ke Docker Hub?

Kubernetes tidak bisa mengambil image dari laptop kamu. Image harus ada di registry (contoh: Docker Hub) supaya cluster bisa menarik dan menjalankannya. Jadi image ini nanti dipakai saat proses deploy di Kubernetes.

---

## 1. Buka Docker Hub

Buka `https://hub.docker.com` dan pastikan kamu sudah login. Kalau belum punya akun, buat dulu di halaman itu.

---

## 2. Login dari terminal

```bash
docker login
```

Masukkan username dan password Docker Hub kamu.

---

## 3. Tag image dengan versi

Ganti `USERNAME` dengan username Docker Hub kamu. Jangan pakai `latest`, gunakan versi.

```bash
docker tag app-post:1.0 USERNAME/app-post:1.0.0
```

---

## 4. Push ke Docker Hub

```bash
docker push USERNAME/app-post:1.0.0
```

---

## 5. Cek di Docker Hub

Masuk ke halaman Docker Hub kamu dan pastikan image `app-post:1.0.0` sudah muncul.

---

## Catatan penting

Image yang kamu push ke Docker Hub **default-nya public**. Kalau ingin private:

- Upgrade akun Docker Hub, atau
- Pakai registry sendiri seperti **Harbor**: [How-to: Harbor](../how-to/008-harbor.md)

---

## Ringkasan

Yang sudah kamu lakukan:

1. Login ke Docker Hub.
2. Tag image dengan versi.
3. Push image ke Docker Hub.

Tahap ini selesai, lanjut ke proses deploy di Kubernetes.

**Langkah selanjutnya:**

- **Deploy app dengan file K8s:** [Tutorial: Deploy App Express dengan File K8s](007-app-express-with-k8s-files.md)
- **Registry private:** [How-to: Harbor](../how-to/008-harbor.md) — opsi registry private selain Docker Hub.
