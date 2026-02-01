# How-to: MetalLB (LoadBalancer di Bare Metal / VPS)

MetalLB memberi Service tipe **LoadBalancer** alamat IP eksternal di cluster yang **tidak** punya cloud load balancer bawaan.

---

## Kapan perlu MetalLB?

**Tidak perlu** memasang MetalLB jika Anda memakai **managed Kubernetes** yang sudah menyediakan LoadBalancer:

- **DigitalOcean** (DOKS)
- **Google GKE**, **AWS EKS**, **Azure AKS**
- **Linode LKE**, **Vultr**, **Scaleway**, dll.

Di layanan itu, `Service type: LoadBalancer` sudah otomatis dapat IP eksternal dari provider.

**Perlu** MetalLB jika cluster Anda jalan di:

- **Bare metal** (server fisik on-prem)
- **VM / VPS** tanpa integrasi load balancer (misalnya K3s di VPS DigitalOcean/Linode tanpa pakai DO Load Balancer)
- **Cluster on-prem** atau di data center yang tidak punya cloud controller

Singkatnya: kalau buat Service `LoadBalancer` dan kolom EXTERNAL-IP tetap `<pending>`, itu saatnya pasang MetalLB (dan siapkan range IP yang boleh dipakai).

---

## 1. Install MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml
```

---

## 2. Cek MetalLB sudah jalan

Pastikan pod MetalLB berjalan di namespace `metallb-system`:

```bash
kubectl get pods -n metallb-system
```

---

## 3. Simpan config IP pool

Buat file `metallb-pool.yaml` (ganti `178.128.91.208/32` dengan range IP yang boleh dipakai di jaringan Anda):

```yaml
# metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
  - 178.128.91.208/32
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: advert
  namespace: metallb-system
```

Simpan ke disk, misalnya:

```bash
# Edit dengan editor atau paste isi di atas ke file metallb-pool.yaml
nano metallb-pool.yaml
```

---

## 4. Apply config IP pool

```bash
kubectl apply -f metallb-pool.yaml
```

---

## 5. Test Load Balancer

Pastikan sudah ada Deployment (misalnya Nginx). Lalu expose dengan tipe LoadBalancer:

```bash
kubectl expose deployment nginx \
  --type=LoadBalancer \
  --port=80 \
  --name=nginx-lb
```

---

## 6. Cek Service

```bash
kubectl get svc nginx-lb
```

**Contoh keluaran (harus ada EXTERNAL-IP):**

```
NAME       TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)        AGE
nginx-lb   LoadBalancer   10.43.x.x      178.128.91.208   80:xxxxx/TCP   xxs
```

Akses via: `http://178.128.91.208` (atau IP yang Anda set di pool).

---

## 7. Hapus Load Balancer (setelah tes)

Setelah berhasil diakses, hapus Service LoadBalancer (dan Deployment jika hanya untuk tes):

```bash
kubectl delete service nginx-lb
kubectl delete deployment nginx
```

Verifikasi:

```bash
kubectl get svc
kubectl get deployment
```
