# Tutorial: Let's Encrypt dengan cert-manager

**Prasyarat:** Selesaikan dulu [Tutorial: Longhorn](001-longhorn.md). Pastikan `kubectl` bisa akses cluster dan Ingress controller aktif (default K3s: Traefik).

**Tujuan:** Memasang cert-manager, membuat `ClusterIssuer` Let's Encrypt, lalu mengaktifkan HTTPS pada Ingress.

---

## Kenapa perlu Let's Encrypt?

Setelah aplikasi berjalan di K3s, biasanya kamu ingin akses lewat **HTTPS** agar data aman dan browser tidak menandai situs sebagai "Not Secure". Let's Encrypt menyediakan sertifikat gratis, tapi prosesnya butuh **otomatisasi** (request, renew, dan simpan ke Secret). Di Kubernetes, otomatisasi ini dilakukan oleh **cert-manager**. Jadi alur belajarnya natural: setelah storage siap → pasang cert-manager agar Ingress bisa punya TLS.

---

## Persiapan

Pastikan:

- Domain sudah mengarah ke IP publik/LoadBalancer.
- Ingress controller aktif (contoh: Traefik bawaan K3s).
- Port 80/443 terbuka di firewall.

Buat folder kerja agar manifest rapi:

```bash
mkdir -p deploy-tools/letsencrypt
cd deploy-tools/letsencrypt
```

---

## 1. Install cert-manager

Pasang manifest resmi:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

Tunggu hingga semua Pod siap:

```bash
kubectl get pods -n cert-manager
```

Minimal ada Pod berikut berstatus **Running**:

```
cert-manager
cert-manager-webhook
cert-manager-cainjector
```

---

## 2. Buat ClusterIssuer (Let's Encrypt)

Simpan sebagai `letsencrypt-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: kamu@email.com # ganti dengan emailmu
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

Terapkan:

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

---

## 3. Buat Ingress dengan TLS

Simpan sebagai `nginx-https.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-https
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - nginx.yourdomain.com # ganti dengan domain kamu
      secretName: nginx-tls # cert disimpan di Secret ini
  rules:
    - host: nginx.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

Terapkan:

```bash
kubectl apply -f nginx-https.yaml
```

---

## 4. Cek sertifikat

```bash
kubectl get certificate
kubectl describe certificate nginx-tls
```

Jika sukses, status akan **Ready** dan Secret `nginx-tls` terisi sertifikat.

---

## Ringkasan

Yang sudah kamu lakukan:

1. Memasang cert-manager di cluster.
2. Membuat `ClusterIssuer` untuk Let's Encrypt.
3. Mengaktifkan TLS di Ingress dan memverifikasi sertifikat.

**Langkah selanjutnya:**

- **How-to cert-manager:** [003 - Let's Encrypt (cert-manager)](../how-to/003-letsencrypt-cert-manager.md) — versi singkat tanpa konteks tutorial.
- **LoadBalancer:** [How-to: MetalLB](../how-to/001-metallb-loadbalancer.md) — jika butuh IP publik di bare metal/VPS.
# Tutorial: HTTPS dengan Let's Encrypt (cert-manager)

**Prasyarat:** Selesaikan [Tutorial: Memulai dengan K3s](000-k3s-getting-started.md) dan [Tutorial: Longhorn](001-longhorn.md). Pastikan Ingress controller (Traefik) aktif, serta domain sudah mengarah ke IP publik/LoadBalancer. Untuk bare metal/VPS, LoadBalancer bisa pakai [MetalLB](../how-to/001-metallb-loadbalancer.md).

**Tujuan:** Mengaktifkan HTTPS otomatis dengan Let's Encrypt menggunakan cert-manager, lalu memasang Ingress TLS untuk aplikasi.

---

## Persiapan

Sebelum mulai:

- Domain sudah mengarah ke IP publik/LoadBalancer.
- Port `80` dan `443` terbuka.
- Traefik aktif (Ingress K3s).
- `kubectl` bisa akses cluster.

Buat folder kerja supaya file manifest rapi:

```bash
mkdir -p letsencrypt
cd letsencrypt
```

---

## 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

---

## 2. Cek Pod cert-manager

```bash
kubectl get pods -n cert-manager
```

Pastikan Pod utama sudah **Running**:

```
cert-manager
cert-manager-webhook
cert-manager-cainjector
```

---

## 3. Buat ClusterIssuer (Let's Encrypt)

Simpan sebagai `letsencrypt-issuer.yaml`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: kamu@email.com # ganti dengan emailmu
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

Terapkan:

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

---

<details>
<summary>Catatan untuk tutorial selanjutnya (klik untuk lihat)</summary>

Di tutorial yang membutuhkan Ingress, kita akan sering memakai pola konfigurasi seperti ini untuk otomasi TLS. Ini contoh saja, sesuaikan `host`, `service`, dan `secretName` dengan aplikasimu.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-https
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
    - hosts:
        - nginx.yourdomain.com # ganti dengan domain kamu
      secretName: nginx-tls # cert disimpan di Secret ini
  rules:
    - host: nginx.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: nginx
                port:
                  number: 80
```

Verifikasi sertifikat setelah apply:

```bash
kubectl get certificate
kubectl describe certificate <nama-secret-tls>
```

- Ganti `<nama-secret-tls>` sesuai `secretName` di Ingress.
- Status **Ready=True** berarti sertifikat sudah terbit.
- Jika masih **Pending**, cek DNS domain dan akses port `80/443`.

</details>

---

## Ringkasan

Yang sudah kamu lakukan:

1. Menginstal cert-manager.
2. Membuat `ClusterIssuer` untuk Let's Encrypt.

**Langkah selanjutnya:**

- **How-to Let's Encrypt:** [003 - Let's Encrypt (cert-manager)](../how-to/003-letsencrypt-cert-manager.md) — versi ringkas tanpa konteks tutorial.
- **Reference Traefik:** [001 - Traefik (Ingress K3s)](../reference/001-traefik.md) — acuan konfigurasi Ingress.
