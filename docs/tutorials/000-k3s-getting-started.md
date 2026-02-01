# Tutorial: Memulai dengan K3s

**Tujuan:** Memasang K3s di satu server, bisa menjalankan `kubectl` tanpa sudo, lalu mencoba deploy contoh (Nginx) dan menghapusnya lagi.

Dalam tutorial ini Anda akan mempelajari langkah demi langkah: persiapan server, instalasi K3s, verifikasi, konfigurasi kubectl, dan contoh deploy Nginx sebagai tes cluster.

---

## Persiapan

Pastikan Anda punya akses SSH ke server Linux (misalnya Ubuntu/Debian).

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

Anda akan melihat status **active (running)** jika berhasil.

---

## Menggunakan cluster

### 4. Cek node dengan sudo

Untuk sementara, gunakan `k3s kubectl`:

```bash
sudo k3s kubectl get nodes
```

Node Anda (biasanya satu) akan tampil dengan status **Ready**.

### 5. Konfigurasi kubectl (tanpa sudo)

Agar bisa memakai `kubectl` tanpa `sudo`:

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
```

Agar pengaturan ini tetap dipakai setiap kali login:

```bash
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
```

### 6. Tes kubectl

Jalankan:

```bash
kubectl get pods -A
```

Jika daftar namespace dan pod tampil (termasuk `kube-system`), cluster K3s Anda siap dipakai.

**Catatan:** K3s sudah menyertakan [Traefik](../reference/001-traefik.md) sebagai Ingress controller bawaan. Untuk cek status atau konfigurasi Traefik, lihat referensi tersebut.

---

## Contoh: Deploy Nginx (tes cluster)

Contoh singkat untuk memastikan cluster bisa menjalankan workload: deploy Nginx, cek akses, lalu hapus lagi.

### 7. Deploy Nginx

Buat Deployment dan Service tipe NodePort:

```bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --type=NodePort --port=80
```

### 8. Cek Service

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

### 9. Akses Nginx

Ganti `IP_SERVER` dengan IP node/server Anda dan `30007` jika port Anda berbeda.

**Dari terminal (curl):**

```bash
curl http://IP_SERVER:30007
```

**Di browser:** buka `http://IP_SERVER:30007` — harus tampil halaman default Nginx (Welcome to nginx!).

### 10. Uninstall Nginx (langkah berikutnya)

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

Anda telah:

1. Memperbarui server dan memasang K3s
2. Memverifikasi layanan K3s
3. Mengatur kubeconfig dan menggunakan `kubectl` tanpa sudo
4. Mencoba deploy Nginx, mengaksesnya, lalu menghapusnya (uninstall)

**Langkah selanjutnya:**

- **Multi-node:** [How-to: Cluster K3s Multi-Node](../how-to/000-k3s-multi-node.md) — menambah worker node.
- **LoadBalancer (bare metal/VPS):** [How-to: MetalLB](../how-to/001-metallb-loadbalancer.md) — agar Service tipe `LoadBalancer` mendapat IP eksternal tanpa cloud provider.
- **Ingress:** [Reference: Traefik](../reference/001-traefik.md) — Ingress controller bawaan K3s, cek status dan konfigurasi.
