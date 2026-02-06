# Tutorial: Memulai dengan K3s

**Tujuan:** Memasang K3s di satu server, bisa menjalankan `kubectl` tanpa sudo, lalu mencoba deploy contoh (Nginx) dan menghapusnya lagi.

Di sini kamu akan mengikuti langkah demi langkah: persiapan server, instalasi K3s, verifikasi, konfigurasi kubectl, dan contoh deploy Nginx sebagai tes cluster.

---

## Kenapa pakai K3s?

K3s adalah distribusi Kubernetes yang **ringan** dan **sederhana** untuk dijalankan di satu server (VPS atau bare metal). Dibanding Kubernetes penuh (kubeadm, kubelet, banyak komponen), K3s cukup **satu binary**, resource (RAM/CPU) lebih kecil, dan instalasinya satu perintah. Cocok untuk **self-hosted**, **edge**, atau cluster kecil yang tidak butuh fitur enterprise.

K3s tetap **kompatibel dengan API Kubernetes** — `kubectl`, Helm, dan manifest YAML yang dipakai di cluster lain bisa dipakai di sini. Jadi kalau kamu mau belajar Kubernetes atau menjalankan workload nyata (web app, database, dll.) di satu server dengan biaya minimal, K3s adalah pilihan yang masuk akal.

---

## Persiapan

Pastikan kamu punya akses SSH ke server Linux (misalnya Ubuntu/Debian).

### 1. Update sistem

Perbarui paket dan reboot jika perlu:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

Setelah reboot, login lagi ke server.

---

## Instalasi K3s

### 2. Pasang K3s

Jalankan skrip instalasi resmi:

```bash
curl -sfL https://get.k3s.io | sh -
```

K3s akan terpasang dan layanan `k3s` berjalan sebagai systemd service.

### 3. Cek status layanan

Pastikan layanan K3s aktif:

```bash
sudo systemctl status k3s
```

Status yang tampil: **active (running)** jika berhasil.

---

## Menggunakan cluster

### 4. Cek node (sementara pakai sudo)

Sebelum `kubectl` siap tanpa `sudo`, gunakan `k3s kubectl`:

```bash
sudo k3s kubectl get nodes
```

Node (biasanya satu) akan tampil dengan status **Ready**.

### 5. Konfigurasi kubectl tanpa sudo

Salin kubeconfig K3s ke home user agar `kubectl` bisa dipakai tanpa `sudo`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
```

### 6. Set KUBECONFIG (kubectl/Helm)

Supaya `kubectl` dan Helm tahu cluster yang benar (dan tidak mencoba `localhost:8080`), arahkan `KUBECONFIG` ke file `~/.kube/config`:

```bash
export KUBECONFIG=~/.kube/config
```

Agar pengaturan ini tetap dipakai setiap kali login:

```bash
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
```

Jika ingin langsung aktif di sesi sekarang:

```bash
source ~/.bashrc
```

Kalau `KUBECONFIG` belum diarahkan, biasanya muncul error seperti ini saat install Helm:

```
Error: INSTALLATION FAILED: Kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp [::1]:8080: connect: connection refused
```

### 7. Tes akses kubectl

Jalankan:

```bash
kubectl get pods -A
```

Jika daftar namespace dan pod tampil (termasuk `kube-system`), cluster K3s kamu siap dipakai.

**Catatan:** K3s sudah menyertakan [Traefik](../reference/001-traefik.md) sebagai Ingress controller bawaan. Untuk cek status atau konfigurasi Traefik, lihat referensi tersebut.

---

## Contoh: Deploy Nginx (tes cluster)

Contoh singkat untuk memastikan cluster bisa menjalankan workload: deploy Nginx, cek akses, lalu hapus lagi.

### 8. Deploy Nginx

Buat Deployment dan Service tipe NodePort:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
```

### 9. Cek Service

Lihat Service dan port NodePort:

```bash
kubectl get svc
```

**Contoh keluaran:**

```
NAME    TYPE       CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
nginx   NodePort   10.43.x.x    <none>        80:30007/TCP    xxs
```

Catat port NodePort (angka setelah `80:`, misalnya `30007`).

### 10. Akses Nginx

Ganti `IP_SERVER` dengan IP node/server kamu dan `30007` jika port kamu berbeda.

**Dari terminal (curl):**

```bash
curl http://IP_SERVER:30007
```

**Di browser:** buka `http://IP_SERVER:30007` — harus tampil halaman default Nginx (Welcome to nginx!).

### 11. Uninstall Nginx (langkah berikutnya)

Setelah berhasil diakses, hapus Service dan Deployment:

```bash
kubectl delete service nginx
kubectl delete deployment nginx
```

Verifikasi sudah bersih:

```bash
kubectl get svc
kubectl get deployment
```

Nginx dan Service-nya tidak lagi muncul di daftar.

---

## Ringkasan

Yang sudah kamu lakukan:

1. Memperbarui server dan memasang K3s
2. Memverifikasi layanan K3s
3. Mengatur kubeconfig dan menggunakan `kubectl` tanpa sudo
4. Mencoba deploy Nginx, mengaksesnya, lalu menghapusnya (uninstall)

**Langkah selanjutnya:**

- **Persistent storage (Longhorn):** [Tutorial: Longhorn](001-longhorn.md) — setelah K3s jalan, pasang Longhorn agar cluster bisa dipakai untuk database atau aplikasi yang butuh data tetap (persistent volume) di self-hosted/VPS.
- **Multi-node:** [How-to: Cluster K3s Multi-Node](../how-to/000-k3s-multi-node.md) — menambah worker node.
- **LoadBalancer (bare metal/VPS):** [How-to: MetalLB](../how-to/001-metallb-loadbalancer.md) — agar Service tipe `LoadBalancer` mendapat IP eksternal tanpa cloud provider.
- **Ingress:** [Reference: Traefik](../reference/001-traefik.md) — Ingress controller bawaan K3s, cek status dan konfigurasi.
