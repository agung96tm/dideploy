# How-to: Harbor

Harbor adalah **container registry**. Ini cocok sebagai alternatif Docker Hub kalau kamu ingin repository **private** tanpa biaya langganan (karena di‑host di server sendiri).

Panduan singkat ini fokus pada pemasangan Harbor di K3s.

## Prasyarat

- Selesai sampai tutorial K3s + Let's Encrypt.
- Longhorn sudah jadi StorageClass default.
- Domain sudah mengarah ke IP publik/LoadBalancer.

## 1. Buat namespace

```bash
kubectl create namespace harbor
```

## 2. Buat file `harbor.yml`

```yaml
expose:
  type: ingress

  tls:
    enabled: true
    certSource: secret
    secret:
      secretName: harbor-tls

  ingress:
    className: traefik
    hosts:
      core: harbor.example.com
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-prod
      traefik.ingress.kubernetes.io/router.entrypoints: websecure

externalURL: https://harbor.example.com

harborAdminPassword: "GANTI_PASSWORD_ADMIN"

persistence:
  enabled: true
  resourcePolicy: keep
  persistentVolumeClaim:
    registry:
      storageClass: longhorn
      size: 30Gi
    jobservice:
      storageClass: longhorn
      size: 1Gi
    database:
      storageClass: longhorn
      size: 5Gi
    redis:
      storageClass: longhorn
      size: 1Gi
    trivy:
      storageClass: longhorn
      size: 5Gi

database:
  type: internal

redis:
  type: internal

trivy:
  enabled: true

metrics:
  enabled: false
```

## 3. Install Harbor

```bash
helm repo add harbor https://helm.goharbor.io
helm repo update

helm install harbor harbor/harbor \
  -n harbor \
  -f values.yaml
```

## 4. Cek PVC

```bash
kubectl get pvc -n harbor
```

Contoh output:

```
NAME                              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-harbor-redis-0               Bound    pvc-a8832956-29a9-4798-a802-fe4a21a79bca   1Gi        RWO            longhorn       <unset>                 13d
data-harbor-trivy-0               Bound    pvc-e7f4ae1f-0ffa-46a5-9eb7-91c6c4c66cce   5Gi        RWO            longhorn       <unset>                 13d
database-data-harbor-database-0   Bound    pvc-b8bd7ba8-c48e-4641-b1fb-11bdf5bd1e61   5Gi        RWO            longhorn       <unset>                 13d
harbor-jobservice                 Bound    pvc-d0514b39-bb16-4c72-af3e-a31df697d9c3   1Gi        RWO            longhorn       <unset>                 13d
harbor-registry                   Bound    pvc-55783a3f-31ac-4117-b3d4-c4dcfa438bd8   20Gi       RWO            longhorn       <unset>                 13d
```

## 5. Akses Harbor

```
https://harbor.example.com
```

Login dengan:
```
Username: admin
Password: sesuai `harborAdminPassword`
```

---

## FAQ singkat

**Apakah `harbor.yml` boleh dihapus?**  
Sebaiknya jangan. File ini berguna saat:

- upgrade Harbor
- ubah ukuran PVC
- mengaktifkan fitur tambahan

**Kenapa password ditaruh di file? Aman?**  
Kurang aman. Kalau mau lebih aman:

1. Hapus baris:

```yaml
harborAdminPassword: ...
```

2. Install dengan `--set`:

```bash
helm install harbor harbor/harbor \
  -n harbor \
  -f harbor.yml \
  --set harborAdminPassword='PASSWORD_SUPER_RAHASIA'
```

Kalau sudah terlanjur, ubah password dari UI Harbor setelah login.