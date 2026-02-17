# Tutorial: Membuat Aplikasi (Express + PostgreSQL + Redis)

**Prasyarat:** Selesaikan dulu [Tutorial: Memulai dengan K3s](000-k3s-getting-started.md) dan [Tutorial: Longhorn](002-longhorn.md). Traefik sudah aktif (bawaan K3s).

**Tujuan:** Membuat API sederhana (CRUD posts) dengan Express.js + Sequelize + PostgreSQL, lalu menambahkan cache Redis untuk endpoint `GET /categories`.

---

## Gambaran singkat

- **PostgreSQL** menyimpan data posts dan categories.
- **Redis** menyimpan cache untuk list categories.
- **Express** jadi API server.

---

## 1. Buat folder app-nya terlebih dahulu

```bash
mkdir -p deploy-tools/app
cd deploy-tools/app
```

---

## 2. Siapkan project Node.js-nya

```bash
npm init -y
npm install express sequelize pg pg-hstore redis dotenv
```

---

## 3. Bikin struktur folder berikut (sekalian Dockerfile)

```
app/
├─ app.js
├─ .env
├─ .gitignore
├─ Dockerfile
└─ models/
   ├─ index.js
   ├─ category.js
   └─ post.js
```

Kalau mau cepat, jalankan perintah ini:

```bash
mkdir -p models
touch app.js .env .gitignore Dockerfile models/index.js models/category.js models/post.js
```

---

## 4. Isi models dengan kodingan berikut

### `models/index.js`

```javascript
const { Sequelize } = require("sequelize");

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASS,
  {
    host: process.env.DB_HOST,
    port: process.env.DB_PORT,
    dialect: "postgres",
    logging: false,
  }
);

module.exports = { sequelize };
```

### `models/post.js`

```javascript
const { DataTypes } = require("sequelize");
const { sequelize } = require("./index");

const Post = sequelize.define("Post", {
  title: { type: DataTypes.STRING, allowNull: false },
  body: { type: DataTypes.TEXT, allowNull: false },
});

module.exports = { Post };
```

### `models/category.js`

```javascript
const { DataTypes } = require("sequelize");
const { sequelize } = require("./index");

const Category = sequelize.define("Category", {
  name: { type: DataTypes.STRING, allowNull: false },
});

module.exports = { Category };
```

### `app.js`

```javascript
require("dotenv").config();
const express = require("express");
const { sequelize } = require("./models");
const { Post } = require("./models/post");
const { Category } = require("./models/category");
const { createClient } = require("redis");

const app = express();
app.use(express.json());

const redisClient = createClient({ url: process.env.REDIS_URL });
redisClient.on("error", (err) => console.error("Redis error", err));

app.get("/health", (_req, res) => res.json({ status: "ok" }));

app.post("/posts", async (req, res) => {
  const post = await Post.create({ title: req.body.title, body: req.body.body });
  res.json(post);
});

app.get("/posts", async (_req, res) => {
  const posts = await Post.findAll();
  res.json(posts);
});

app.get("/categories", async (_req, res) => {
  const cacheKey = "cache:categories";
  const cached = await redisClient.get(cacheKey);
  if (cached) {
    return res.json(JSON.parse(cached));
  }

  const categories = await Category.findAll();
  await redisClient.setEx(cacheKey, 60, JSON.stringify(categories));
  res.json(categories);
});

app.get("/posts/:id", async (req, res) => {
  const post = await Post.findByPk(req.params.id);
  if (!post) return res.status(404).json({ message: "Not found" });
  res.json(post);
});

app.put("/posts/:id", async (req, res) => {
  const post = await Post.findByPk(req.params.id);
  if (!post) return res.status(404).json({ message: "Not found" });
  post.title = req.body.title ?? post.title;
  post.body = req.body.body ?? post.body;
  await post.save();
  res.json(post);
});

app.delete("/posts/:id", async (req, res) => {
  const post = await Post.findByPk(req.params.id);
  if (!post) return res.status(404).json({ message: "Not found" });
  await post.destroy();
  res.json({ message: "Deleted" });
});

const start = async () => {
  await redisClient.connect();
  await sequelize.authenticate();
  await sequelize.sync();
  app.listen(process.env.APP_PORT || 3000, () => {
    console.log("API running");
  });
};

start();
```

---

## 5. Isi `.env`

```env
APP_PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=appdb
DB_USER=appuser
DB_PASS=appsecret
REDIS_URL=redis://localhost:6379/0
```

Sesuaikan valuenya dengan DB/Redis yang kamu pakai.

---

## 6. Isi `.gitignore`

```gitignore
node_modules
.env
```

---

## 7. Isi Dockerfile

```Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["node", "app.js"]
```

---

## 8. Build image (cukup build saja)

```bash
docker build -t app-post:1.0 .
```

---

<details>
<summary>Opsional: jalankan dengan Docker Compose (tanpa build)</summary>

Biar yakin app-nya jalan, kamu bisa coba lewat Docker Compose dulu.

Simpan sebagai `docker-compose.yml`:

```yaml
version: "3.9"
services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      APP_PORT: "3000"
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASS: appsecret
      REDIS_URL: redis://redis:6379/0
    depends_on:
      - db
      - redis

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: appsecret
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
```

Struktur folder:

```
app/
├─ app.js
├─ .env
├─ .gitignore
├─ Dockerfile
├─ docker-compose.yml
└─ models/
   ├─ index.js
   ├─ category.js
   └─ post.js
```

Jalankan:

```bash
docker compose up -d
```

Tes aksesnya:

```
http://localhost:3000/health
```

Kalau sudah oke, stop dengan:

```bash
docker compose down
# docker compose down -v -- hapus dengan volume-nya --
```

</details>

---

## Ringkasan

Yang sudah kamu lakukan:

1. Membuat API sederhana dengan Express + Sequelize.
2. Menambahkan cache Redis untuk contoh endpoint.
3. Menyiapkan cache Redis di endpoint `GET /categories`.

**Langkah selanjutnya:**

- **Push image ke Docker Hub:** [Tutorial: Push Image ke Docker Hub](006-hub-docker-and-push-image.md) — agar image bisa dipakai di Kubernetes.
- **Tutorial Ingress Controller:** [Ingress Controller (Traefik)](004-ingress-controller.md) — pahami alur Ingress sebelum pasang domain.