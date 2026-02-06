# How-to: Longhorn (Persistent Storage)

Longhorn adalah sistem **persistent storage** berbasis block storage untuk Kubernetes. Longhorn membuat volume bisa dipakai ulang oleh Pod (misalnya data database atau file upload) dan cocok dipakai di cluster kecil maupun single-node (termasuk VPS).

**Kapan dipakai:** saat aplikasi butuh penyimpanan yang tetap (persistent), misalnya database, cache, atau file yang harus tidak hilang saat Pod di-restart atau diganti.

---

## Prasyarat

Sebelum instalasi:

- Pastikan server punya **storage cukup** (Longhorn butuh minimal ~10GB ruang kosong).
- Pastikan kamu punya akses ke namespace `longhorn-system` untuk pengecekan/manajemen.

Cek node dan kapasitas disk:

```bash
kubectl get nodes
df -h
```

---

## 1. Instalasi Longhorn

Pasang Longhorn lewat manifest resmi:

```bash
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/v1.6.2/deploy/longhorn.yaml
```

Tunggu sampai semua komponen siap (bisa memakan beberapa menit).

---

## 2. Cek Pod Longhorn

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

## 3. Jadikan Longhorn sebagai StorageClass default

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

## 4. Uji coba: membuat Persistent Volume

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

- **Longhorn bisa dipakai di VPS single-node** dan aman untuk dipakai di lingkungan kecil.
- **Cocok untuk** database, cache, atau aplikasi lain yang butuh persistent storage.
- **Tidak wajib** untuk aplikasi yang benar-benar stateless (misalnya Nginx tanpa data yang perlu disimpan).
