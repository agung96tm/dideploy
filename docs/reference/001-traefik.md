# Traefik (Ingress Default K3s)

K3s memasang **Traefik** sebagai Ingress controller secara default. Tidak perlu instalasi tambahan.

---

## Cek Traefik sudah jalan

Pastikan pod Traefik berjalan di namespace `kube-system`:

```bash
kubectl get pods -n kube-system | grep traefik
```

**Contoh keluaran:**

```
traefik-xxxxx   Running
```

Jika status **Running**, Traefik siap dipakai untuk Ingress.
