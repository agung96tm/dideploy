# How-to: Adminer

Panduan ringkas untuk menjalankan Adminer di K3s.

## Prasyarat

- Ingress controller aktif (contoh: Traefik bawaan K3s).
- Jika pakai TLS, ikuti [How-to: Let's Encrypt (cert-manager)](003-letsencrypt-cert-manager.md).

**Catatan singkat:** jika nanti menambahkan kredensial ke Secret, jangan commit ke Git. Apply Secret dari mesin lokal yang aman atau lewat CI/CD secret.

## 1. Buat namespace (sekali saja)

```bash
kubectl get ns tools || kubectl create ns tools
```

## 2. Buat `adminer.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: adminer
  namespace: tools
  labels:
    app: adminer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: adminer
  template:
    metadata:
      labels:
        app: adminer
    spec:
      containers:
        - name: adminer
          image: adminer:5.4.1
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ADMINER_DEFAULT_SERVER
              value: "db-service.default.svc.cluster.local" # optional
---
apiVersion: v1
kind: Service
metadata:
  name: adminer
  namespace: tools
spec:
  type: ClusterIP
  selector:
    app: adminer
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: adminer
  namespace: tools
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: traefik
  rules:
    - host: adminer.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: adminer
                port:
                  number: 80
  tls:
    - hosts:
        - adminer.example.com
      secretName: adminer-tls
```

Terapkan:

```bash
kubectl apply -f adminer.yaml
```

## Penjelasan variabel

- `ADMINER_DEFAULT_SERVER`: host default database yang otomatis terisi saat login (opsional).
- `adminer.example.com`: ganti dengan domain milikmu.
- `adminer-tls`: nama Secret TLS dari cert-manager untuk domain tersebut.
