# How-to: Cluster K3s Multi-Node

Panduan singkat menambah worker node ke cluster K3s (1 control-plane + N worker).

## Prasyarat

- K3s sudah terpasang di satu server sebagai **control-plane**.
- **1 server node** (control-plane) dan **N worker node** (Linux).
- Akses SSH ke semua node.

---

## 1. Di server node (control-plane)

### Ambil token join

Token dipakai worker untuk bergabung ke cluster:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Salin token yang tampil (string panjang). Simpan untuk langkah di worker.

### Catat IP server

Di server node, cek alamat IP yang dipakai worker untuk mengakses API (biasanya IP LAN):

```bash
ip a
```

Contoh: `192.168.1.10`. Ganti dengan IP server kamu di langkah worker.

---

## 2. Di setiap worker node

Di **setiap** mesin yang akan jadi worker:

1. Ganti `IP_SERVER` dengan IP server (misalnya `192.168.1.10`)
2. Ganti `TOKEN_DARI_SERVER` dengan token dari Langkah 1

Lalu jalankan:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://IP_SERVER:6443 \
  K3S_TOKEN=TOKEN_DARI_SERVER \
  sh -
```

Contoh nyata:

```bash
curl -sfL https://get.k3s.io | \
  K3S_URL=https://192.168.1.10:6443 \
  K3S_TOKEN=K10abc123... \
  sh -
```

Tunggu sampai instalasi selesai. Layanan `k3s-agent` akan berjalan di worker.

---

## 3. Verifikasi

Kembali ke **server node** (control-plane). Pastikan `KUBECONFIG` sudah di-set (lihat [Memulai dengan K3s](../tutorials/000-k3s-getting-started.md)), lalu:

```bash
kubectl get nodes
```

Contoh keluaran:

```
NAME        STATUS   ROLES                  AGE   VERSION
server-1    Ready    control-plane,master   10m   v1.xx.x+k3s
worker-1    Ready    <none>                  5m   v1.xx.x+k3s
worker-2    Ready    <none>                  3m   v1.xx.x+k3s
```

Semua node dengan status **Ready** berarti cluster multi-node kamu sudah berjalan.

---

## Troubleshooting singkat

| Masalah | Cek |
|--------|-----|
| Worker tidak muncul | Firewall: pastikan port **6443** (API) dari worker ke server terbuka |
| Token salah | Jalankan lagi `sudo cat /var/lib/rancher/k3s/server/node-token` di server |
| IP salah | Pastikan worker bisa ping ke IP server (`ping IP_SERVER`) |
