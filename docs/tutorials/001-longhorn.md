# Tutorial: Longhorn (Persistent Storage)

**Prasyarat:** Selesaikan dulu [Tutorial: Memulai dengan K3s](000-k3s-getting-started.md) — K3s sudah terpasang dan `kubectl` berjalan normal.

**Tujuan:** Memasang Longhorn di cluster K3s (self-hosted/VPS), menjadikannya StorageClass default, lalu menguji persistent volume dengan PVC dan Pod.

---

## Kenapa harus install Longhorn?

Di [tutorial K3s](000-k3s-getting-started.md) kita hanya deploy Nginx stateless: Pod bisa dihapus dan dibuat ulang tanpa kehilangan data karena tidak ada data yang disimpan. Di lingkungan **self-hosted** (VPS atau bare metal), begitu kamu mau jalankan aplikasi yang butuh **data tetap** — misalnya database (PostgreSQL, MySQL), cache (Redis), atau file upload — data itu harus disimpan di **persistent volume**. Tanpa itu, data akan hilang saat Pod di-restart, dijadwal ulang ke node lain, atau dihapus.

K3s single-node **tidak menyediakan StorageClass siap pakai** untuk persistent volume seperti di cloud (misalnya AWS EBS, GCP PD). **Longhorn** menambah kemampuan itu: ia membuat block storage di dalam cluster sehingga PVC (PersistentVolumeClaim) bisa dipakai untuk database, cache, atau file yang harus tetap ada. Jadi alur belajarnya natural: setelah K3s jalan → pasang Longhorn agar cluster siap dipakai untuk workload yang butuh persistent storage.

---

## Persiapan

Sebelum instalasi:

- Pastikan server punya **storage cukup** (Longhorn butuh minimal ~10GB ruang kosong).
- Pastikan kamu punya akses ke namespace `longhorn-system` untuk pengecekan/manajemen.

Cek node dan kapasitas disk:

```bash
kubectl get nodes
df -h
```

---

## Instalasi Longhorn

Pasang Longhorn lewat manifest resmi:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.2/deploy/longhorn.yaml
```

Tunggu sampai semua komponen siap (bisa memakan beberapa menit).

---

## Cek Pod Longhorn

Pastikan semua Pod di namespace `longhorn-system` berstatus **Running**:

```bash
kubectl get pods -n longhorn-system
```

Contoh output yang sehat:

```
NAME                                     READY   STATUS    RESTART   AGE
longhorn-frontend-xxx                     1/1     Running   0         5m
longhorn-manager-xxx                     1/1     Running   0         5m
longhorn-driver-xxx                      1/1     Running   0         5m
...
```

---

## Jadikan Longhorn sebagai StorageClass default

Agar PVC baru otomatis pakai Longhorn tanpa harus menyebut nama StorageClass:

```bash
kubectl patch storageclass longhorn \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

Verifikasi:

```bash
kubectl get storageclass
```

`longhorn` harus tampil dengan kolom **DEFAULT** bernilai `true`:

```
NAME       PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE   DEFAULT
longhorn   driver.longhorn.io      Delete          Immediate           true
```

---

## Uji coba: membuat Persistent Volume

### 1. Buat PersistentVolumeClaim (PVC)

Buat file `test-pvc.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Terapkan:

```bash
kubectl apply -f test-pvc.yaml
```

### 2. Cek PVC dan PV

```bash
kubectl get pvc
kubectl get pv
```

- PVC status **Bound** → Longhorn sudah membuat volume dan mengikatnya.
- PV akan otomatis dibuat oleh Longhorn (provisioner).

### 3. Pakai PVC di Pod

Buat file `nginx-pv.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pv
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /usr/share/nginx/html
          name: webdata
  volumes:
    - name: webdata
      persistentVolumeClaim:
        claimName: test-pvc
```

Terapkan dan cek:

```bash
kubectl apply -f nginx-pv.yaml
kubectl exec -it nginx-pv -- /bin/bash
```

Di dalam Pod, cek folder `/usr/share/nginx/html`. File yang disimpan di sini akan tetap ada meskipun Pod dihapus dan dibuat ulang (selama PVC tidak dihapus).

### 4. Hapus resource uji coba

Setelah selesai uji coba, hapus Pod lalu PVC. Urutan penting: hapus dulu Pod yang memakai PVC, baru hapus PVC.

```bash
kubectl delete pod nginx-pv
kubectl delete pvc test-pvc
```

PV akan otomatis terhapus oleh Longhorn (karena reclaim policy `Delete`). Cek sampai bersih:

```bash
kubectl get pvc
kubectl get pv
kubectl get pods
```

---

## Ringkasan

Yang sudah kamu lakukan:

1. Memasang Longhorn di cluster K3s (self-hosted).
2. Menjadikan Longhorn sebagai StorageClass default.
3. Menguji persistent volume dengan PVC dan Pod, lalu membersihkan resource uji coba.

**Langkah selanjutnya:**

- **How-to Longhorn:** [002 - Longhorn (Persistent Storage)](../how-to/002-longhorn.md) — referensi singkat yang sama, tanpa konteks tutorial.
- **Multi-node:** [How-to: Cluster K3s Multi-Node](../how-to/000-k3s-multi-node.md) — menambah worker node.
- **LoadBalancer:** [How-to: MetalLB](../how-to/001-metallb-loadbalancer.md) — Service tipe LoadBalancer di bare metal/VPS.
