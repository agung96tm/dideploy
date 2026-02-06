# How-to: GlitchTip

Contoh minimal GlitchTip di K3s. Contoh ini mengasumsikan PostgreSQL dan Redis sudah tersedia (external).

## Prasyarat

- PostgreSQL 14+ siap dipakai.
- Redis/Valkey siap dipakai.
- Ingress controller aktif (contoh: Traefik bawaan K3s).
- Jika pakai TLS, ikuti [How-to: Let's Encrypt (cert-manager)](003-letsencrypt-cert-manager.md).

## Catatan keamanan

Jangan commit `glitchtip-secrets.yaml` ke Git. Apply Secret dari mesin lokal yang aman atau lewat CI/CD secret.

## 1. Buat namespace (sekali saja)

```bash
kubectl get ns tools || kubectl create ns tools
```

## 2. Buat `glitchtip-secrets.yaml` (sensitif)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: glitchtip-secrets
  namespace: tools
type: Opaque
stringData:
  SECRET_KEY: "ganti-dengan-random-string"
  DATABASE_URL: "postgres://user:password@postgres-host:5432/glitchtip"
  REDIS_URL: "redis://redis-host:6379/0"
```

## 3. Buat `glitchtip.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: glitchtip
  namespace: tools
  labels:
    app: glitchtip
spec:
  replicas: 1
  selector:
    matchLabels:
      app: glitchtip
  template:
    metadata:
      labels:
        app: glitchtip
    spec:
      containers:
        - name: glitchtip
          image: glitchtip/glitchtip:latest
          ports:
            - containerPort: 8000
              name: http
          env:
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: glitchtip-secrets
                  key: SECRET_KEY
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: glitchtip-secrets
                  key: DATABASE_URL
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: glitchtip-secrets
                  key: REDIS_URL
            - name: TZ
              value: Asia/Jakarta
---
apiVersion: v1
kind: Service
metadata:
  name: glitchtip-service
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: glitchtip
  ports:
    - name: http
      port: 80
      targetPort: 8000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: glitchtip-ingress
  namespace: tools
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - glitchtip.example.com
      secretName: glitchtip-tls
  rules:
    - host: glitchtip.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: glitchtip-service
                port:
                  number: 80
```

Terapkan:

```bash
kubectl apply -f glitchtip-secrets.yaml
kubectl apply -f glitchtip.yaml
```

## Penjelasan variabel (sensitif)

- `SECRET_KEY`: kunci rahasia aplikasi (harus random dan panjang).
- `DATABASE_URL`: URL koneksi PostgreSQL untuk GlitchTip.
- `REDIS_URL`: URL koneksi Redis/Valkey.

## Penjelasan variabel lain

- `glitchtip.example.com`: ganti dengan domain milikmu.
