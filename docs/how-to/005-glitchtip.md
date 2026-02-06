# How-to: GlitchTip

Contoh GlitchTip di K3s, termasuk PostgreSQL dan Redis internal.

## Prasyarat

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
  POSTGRES_DB: "glitchtip"
  POSTGRES_USER: "glitchtip"
  POSTGRES_PASSWORD: "ganti-password-db"
  DATABASE_URL: "postgres://glitchtip:ganti-password-db@glitchtip-postgres.tools.svc.cluster.local:5432/glitchtip"
  REDIS_URL: "redis://glitchtip-redis.tools.svc.cluster.local:6379/0"
```

## 3. Buat `glitchtip-postgres.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: glitchtip-postgres-pvc
  namespace: tools
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: glitchtip-postgres
  namespace: tools
  labels:
    app: glitchtip-postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: glitchtip-postgres
  template:
    metadata:
      labels:
        app: glitchtip-postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: glitchtip-secrets
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: glitchtip-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: glitchtip-secrets
                  key: POSTGRES_PASSWORD
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: pg-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: pg-data
          persistentVolumeClaim:
            claimName: glitchtip-postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: glitchtip-postgres
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: glitchtip-postgres
  ports:
    - port: 5432
      targetPort: 5432
```

## 4. Buat `glitchtip-redis.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: glitchtip-redis
  namespace: tools
  labels:
    app: glitchtip-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: glitchtip-redis
  template:
    metadata:
      labels:
        app: glitchtip-redis
    spec:
      containers:
        - name: redis
          image: redis:7
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: glitchtip-redis
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: glitchtip-redis
  ports:
    - port: 6379
      targetPort: 6379
```

## 5. Buat `glitchtip.yaml`

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
kubectl apply -f glitchtip-postgres.yaml
kubectl apply -f glitchtip-redis.yaml
kubectl apply -f glitchtip.yaml
```

## Penjelasan variabel (sensitif)

- `SECRET_KEY`: kunci rahasia aplikasi (harus random dan panjang).
- `POSTGRES_USER` / `POSTGRES_PASSWORD` / `POSTGRES_DB`: kredensial database internal.
- `DATABASE_URL`: URL koneksi PostgreSQL (Service internal).
- `REDIS_URL`: URL koneksi Redis/Valkey (Service internal).

## Penjelasan variabel lain

- `glitchtip.example.com`: ganti dengan domain milikmu.
