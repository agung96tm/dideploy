# How-to: Metabase

Contoh manifest Metabase + PostgreSQL di K3s.

## Prasyarat

- StorageClass sudah siap (misal Longhorn) untuk PVC.
- Ingress controller aktif (contoh: Traefik bawaan K3s).
- Jika pakai TLS, ikuti [How-to: Let's Encrypt (cert-manager)](003-letsencrypt-cert-manager.md).

## Catatan keamanan

Jangan commit `metabase-secrets.yaml` ke Git. Apply Secret dari mesin lokal yang aman atau lewat CI/CD secret.

## 1. Buat namespace (sekali saja)

```bash
kubectl get ns tools || kubectl create ns tools
```

## 2. Buat `metabase-secrets.yaml` (sensitif)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: metabase-secrets
  namespace: tools
type: Opaque
stringData:
  POSTGRES_DB: "metabase"
  POSTGRES_USER: "metabase"
  POSTGRES_PASSWORD: "ganti-password-db"
  MB_DB_DBNAME: "metabase"
  MB_DB_USER: "metabase"
  MB_DB_PASS: "ganti-password-db"
```

## 3. Buat `metabase-postgres.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: metabase-postgres-pvc
  namespace: tools
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metabase-postgres
  namespace: tools
  labels:
    app: metabase-postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: metabase-postgres
  template:
    metadata:
      labels:
        app: metabase-postgres
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
                  name: metabase-secrets
                  key: POSTGRES_DB
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: metabase-secrets
                  key: POSTGRES_USER
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: metabase-secrets
                  key: POSTGRES_PASSWORD
            - name: TZ
              value: Asia/Jakarta
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: pg-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: pg-data
          persistentVolumeClaim:
            claimName: metabase-postgres-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: metabase-postgres
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: metabase-postgres
  ports:
    - port: 5432
      targetPort: 5432
```

## 4. Buat `metabase.yaml`

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: metabase-pvc
  namespace: tools
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metabase
  namespace: tools
  labels:
    app: metabase
spec:
  replicas: 1
  selector:
    matchLabels:
      app: metabase
  template:
    metadata:
      labels:
        app: metabase
    spec:
      containers:
        - name: metabase
          image: metabase/metabase:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: MB_DB_TYPE
              value: postgres
            - name: MB_DB_DBNAME
              valueFrom:
                secretKeyRef:
                  name: metabase-secrets
                  key: MB_DB_DBNAME
            - name: MB_DB_PORT
              value: "5432"
            - name: MB_DB_USER
              valueFrom:
                secretKeyRef:
                  name: metabase-secrets
                  key: MB_DB_USER
            - name: MB_DB_PASS
              valueFrom:
                secretKeyRef:
                  name: metabase-secrets
                  key: MB_DB_PASS
            - name: MB_DB_HOST
              value: metabase-postgres
            - name: MB_JETTY_PORT
              value: "3000"
            - name: TZ
              value: Asia/Jakarta
          volumeMounts:
            - name: metabase-data
              mountPath: /metabase-data
      volumes:
        - name: metabase-data
          persistentVolumeClaim:
            claimName: metabase-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: metabase-service
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: metabase
  ports:
    - name: http
      port: 80
      targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: metabase-ingress
  namespace: tools
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - metabase.example.com
      secretName: metabase-tls
  rules:
    - host: metabase.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: metabase-service
                port:
                  number: 80
```

Terapkan:

```bash
kubectl apply -f metabase-secrets.yaml
kubectl apply -f metabase-postgres.yaml
kubectl apply -f metabase.yaml
```

## Penjelasan variabel (sensitif)

- `POSTGRES_PASSWORD`: password database PostgreSQL (untuk container Postgres).
- `MB_DB_PASS`: password yang dipakai Metabase untuk konek ke PostgreSQL.
- `POSTGRES_DB`, `POSTGRES_USER`, `MB_DB_DBNAME`, `MB_DB_USER`: nama DB dan user (bisa sama).

## Penjelasan variabel lain

- `MB_DB_HOST`: nama Service PostgreSQL.
- `MB_DB_PORT`: port PostgreSQL.
- `MB_JETTY_PORT`: port internal Metabase.
- `metabase.example.com`: ganti dengan domain milikmu.
