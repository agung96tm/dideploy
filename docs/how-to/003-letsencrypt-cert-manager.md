# How-to: Let's Encrypt (cert-manager)

Panduan singkat untuk menerbitkan sertifikat Let's Encrypt di K3s dengan cert-manager.

## Prasyarat

- Domain sudah mengarah ke IP publik/LoadBalancer.
- Ingress controller aktif (contoh: Traefik bawaan K3s).
- Port 80/443 terbuka.

## 1. Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

```bash
kubectl get pods -n cert-manager
```

## 2. Buat ClusterIssuer

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

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

## 3. Tambah TLS di Ingress

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
      secretName: nginx-tls
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

```bash
kubectl apply -f nginx-https.yaml
kubectl get certificate
```
