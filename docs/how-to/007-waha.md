# How-to: WAHA

Contoh manifest WAHA di K3s.

## Prasyarat

- StorageClass sudah siap (misal Longhorn) untuk PVC.
- Ingress controller aktif (contoh: Traefik bawaan K3s).
- Jika pakai TLS, ikuti [How-to: Let's Encrypt (cert-manager)](003-letsencrypt-cert-manager.md).

## Catatan keamanan

Jangan commit `waha-secrets.yaml` ke Git. Apply Secret dari mesin lokal yang aman atau lewat CI/CD secret.

## 1. Buat namespace (sekali saja)

```bash
kubectl get ns tools || kubectl create ns tools
```

## 2. Buat `waha-secrets.yaml` (sensitif)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: waha-secrets
  namespace: tools
type: Opaque
stringData:
  WAHA_API_KEY_PLAIN: "ganti-api-key-plain"
  WAHA_API_KEY: "sha512:ganti-hash-api-key"
  WAHA_DASHBOARD_USERNAME: "admin"
  WAHA_DASHBOARD_PASSWORD: "ganti-password-dashboard"
  WHATSAPP_SWAGGER_USERNAME: "admin"
  WHATSAPP_SWAGGER_PASSWORD: "ganti-password-swagger"
```

## 3. Buat `waha.yaml`

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: waha-pvc
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
  name: waha
  namespace: tools
  labels:
    app: waha
spec:
  replicas: 1
  selector:
    matchLabels:
      app: waha
  template:
    metadata:
      labels:
        app: waha
    spec:
      containers:
        - name: waha
          image: devlikeapro/waha:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 3000
              name: http
          env:
            - name: WAHA_API_KEY_PLAIN
              valueFrom:
                secretKeyRef:
                  name: waha-secrets
                  key: WAHA_API_KEY_PLAIN
            - name: WAHA_API_KEY
              valueFrom:
                secretKeyRef:
                  name: waha-secrets
                  key: WAHA_API_KEY
            - name: WAHA_RESTART_ALL_SESSIONS
              value: "false"
            - name: WAHA_AUTO_START_DELAY_SECONDS
              value: "5"
            - name: WHATSAPP_RESTART_ALL_SESSIONS
              value: "false"
            - name: TZ
              value: "Asia/Jakarta"
            - name: WAHA_DASHBOARD_ENABLED
              value: "true"
            - name: WAHA_DASHBOARD_USERNAME
              valueFrom:
                secretKeyRef:
                  name: waha-secrets
                  key: WAHA_DASHBOARD_USERNAME
            - name: WAHA_DASHBOARD_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: waha-secrets
                  key: WAHA_DASHBOARD_PASSWORD
            - name: WHATSAPP_SWAGGER_ENABLED
              value: "true"
            - name: WHATSAPP_SWAGGER_USERNAME
              valueFrom:
                secretKeyRef:
                  name: waha-secrets
                  key: WHATSAPP_SWAGGER_USERNAME
            - name: WHATSAPP_SWAGGER_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: waha-secrets
                  key: WHATSAPP_SWAGGER_PASSWORD
          volumeMounts:
            - name: waha-data
              mountPath: /app/.sessions
      volumes:
        - name: waha-data
          persistentVolumeClaim:
            claimName: waha-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: waha-service
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: waha
  ports:
    - name: http
      port: 80
      targetPort: 3000
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: waha-ingress
  namespace: tools
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - waha.example.com
      secretName: waha-tls
  rules:
    - host: waha.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: waha-service
                port:
                  number: 80
```

Terapkan:

```bash
kubectl apply -f waha-secrets.yaml
kubectl apply -f waha.yaml
```

## Penjelasan variabel (sensitif)

- `WAHA_API_KEY_PLAIN`: API key dalam teks biasa untuk akses API.
- `WAHA_API_KEY`: API key versi hash (sha512).
- `WAHA_DASHBOARD_USERNAME` / `WAHA_DASHBOARD_PASSWORD`: kredensial dashboard WAHA.
- `WHATSAPP_SWAGGER_USERNAME` / `WHATSAPP_SWAGGER_PASSWORD`: kredensial halaman Swagger.

## Penjelasan variabel lain

- `WAHA_RESTART_ALL_SESSIONS`: restart semua sesi saat boot (true/false).
- `WAHA_AUTO_START_DELAY_SECONDS`: delay auto-start sesi dalam detik.
- `WHATSAPP_RESTART_ALL_SESSIONS`: restart semua sesi WhatsApp saat boot.
- `waha.example.com`: ganti dengan domain milikmu.
