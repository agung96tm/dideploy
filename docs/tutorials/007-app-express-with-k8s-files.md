# Tutorial: Menyiapkan File K8s untuk App Express

**Prasyarat:** Selesaikan dulu [Tutorial: Push Image ke Docker Hub](006-hub-docker-and-push-image.md). Image aplikasi sudah ada di registry.

**Tujuan:** Menyiapkan file K8s untuk app, database, dan redis. Deployment ke cluster akan dibahas di tutorial berikutnya.

---

## Gambaran singkat

Kita akan bikin file K8s terpisah untuk:

- database (PostgreSQL)
- redis
- app
- service
- ingress

---

## 1. Buat folder deployments

```bash
mkdir -p app/deployments/k8s
```

## 2. Buat `database-secrets.yml`

Simpan di `app/deployments/k8s/database-secrets.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-secrets
type: Opaque
stringData:
  POSTGRES_DB: "appdb"
  POSTGRES_USER: "appuser"
  POSTGRES_PASSWORD: "ganti-password-db"
```

## 3. Buat `database.yml`

Simpan di `app/deployments/k8s/database.yml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-database-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: longhorn
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-database
  labels:
    app: app-database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-database
  template:
    metadata:
      labels:
        app: app-database
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: database-secrets
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: pg-data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: pg-data
          persistentVolumeClaim:
            claimName: app-database-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: app-database
spec:
  type: ClusterIP
  selector:
    app: app-database
  ports:
    - port: 5432
      targetPort: 5432
```

## 4. Buat `redis.yml`

Simpan di `app/deployments/k8s/redis.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-redis
  labels:
    app: app-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-redis
  template:
    metadata:
      labels:
        app: app-redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: app-redis
spec:
  type: ClusterIP
  selector:
    app: app-redis
  ports:
    - port: 6379
      targetPort: 6379
```

## 5. Buat `app-secrets.yaml`

Simpan di `app/deployments/k8s/app-secrets.yaml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  DB_NAME: "appdb"
  DB_USER: "appuser"
  DB_PASS: "ganti-password-db"
  DB_HOST: "app-database"
  DB_PORT: "5432"
  REDIS_URL: "redis://app-redis:6379/0"
  APP_PORT: "3000"
```

## 6. Buat `app-deployment.yml`

Simpan di `app/deployments/k8s/app-deployment.yml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-posts
  labels:
    app: app-posts
spec:
  replicas: 1
  selector:
    matchLabels:
      app: app-posts
  template:
    metadata:
      labels:
        app: app-posts
    spec:
      containers:
        - name: app
          image: USERNAME/app-post:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          envFrom:
            - secretRef:
                name: app-secrets
```

## 7. Buat `app-service.yml`

Simpan di `app/deployments/k8s/app-service.yml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-posts
spec:
  type: ClusterIP
  selector:
    app: app-posts
  ports:
    - port: 80
      targetPort: 3000
```

## 8. Buat `app-ingress.yml`

Simpan di `app/deployments/k8s/app-ingress.yml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-posts-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-posts
                port:
                  number: 80
```

## Struktur folder akhir

```
app/
├─ app.js
├─ .env
├─ .gitignore
├─ Dockerfile
├─ docker-compose.yml
├─ models/
│  ├─ index.js
│  ├─ category.js
│  └─ post.js
└─ deployments/
   └─ k8s/
      ├─ database-secrets.yml
      ├─ database.yml
      ├─ redis.yml
      ├─ app-secrets.yaml
      ├─ app-deployment.yml
      ├─ app-service.yml
      └─ app-ingress.yml
```

---

## 3. Update image app

Di `app-deployment.yml`, ganti:

```
image: USERNAME/app-post:1.0.0
```

sesuai image yang kamu push ke Docker Hub.

---

## Ringkasan

Yang sudah kamu lakukan:

1. Menyiapkan file K8s di `app/deployments/k8s/`.
2. Mengganti image aplikasi ke versi yang sudah di-push.

**Langkah selanjutnya:**

- **Tutorial deploy via GitHub Actions:** (akan dibahas di tutorial berikutnya).
