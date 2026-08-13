# PH Healthcare — Docker Local Development Setup Guide

> **এই ফাইলটা কীসের জন্য?**
> এই ডকুমেন্টে **PH Healthcare** প্রজেক্টের ব্যাকএন্ড (`backend/` — Node.js + Express + Prisma) এবং ফ্রন্টএন্ড (`frontend/` — Next.js + Bun) অ্যাপকে **লোকাল মেশিনে Docker দিয়ে ডেভেলপমেন্ট মোডে** কীভাবে রান করতে হয়, তার সম্পূর্ণ, ধাপে-ধাপে গাইড দেওয়া আছে। Network তৈরি থেকে শুরু করে database, backend, frontend কন্টেইনার, Prisma migration, এবং সচরাচর হওয়া সব এররের সমাধান — সব এখানে একসাথে সাজানো।
>
> ভবিষ্যতে প্রজেক্ট নতুন করে সেটআপ করতে হলে, বা কোনো এরর আবার দেখা দিলে, এই ফাইলটাই প্রথমে দেখুন।

---

## 📁 প্রজেক্ট স্ট্রাকচার (রেফারেন্সের জন্য)

```
ph-helth/
├── backend/          # Node.js + Express + Prisma
│   ├── Dockerfile
│   ├── .env
│   └── ...
└── frontend/         # Next.js + Bun
    ├── Dockerfile
    ├── .env
    └── ...
```

> ⚠️ পুরনো নোট/স্ক্রিনশটে ফোল্ডারের নাম `server` ও `client` লেখা থাকতে পারে — বর্তমান প্রজেক্টে আসল নাম হলো **`backend`** ও **`frontend`**। কমান্ড কপি করার সময় এই পার্থক্য খেয়াল রাখবেন।

সব কমান্ড প্রজেক্ট রুট (`ph-helth/`) থেকে PowerShell-এ চালাতে হবে, যদি না আলাদা করে বলা থাকে।

---

## 1. Docker Network তৈরি করা

Backend, frontend, database কন্টেইনার একে অপরকে নাম দিয়ে (hostname হিসেবে) খুঁজে পেতে হলে সবাইকে একই কাস্টম নেটওয়ার্কে থাকতে হয়।

```powershell
docker network create ph-net
```

চেক করতে:

```powershell
docker network ls
docker network inspect ph-net
```

---

## 2. Docker Volumes তৈরি করা

| Volume নাম             | কী কাজ করে                                          |
|--------------------------|--------------------------------------------------------|
| `ph-pg-data`              | PostgreSQL-এর ডেটা persist করার জন্য                  |
| `server-node-modules`     | Backend-এর `node_modules`                              |
| `server-logs`             | Backend-এর লগ ফাইল                                     |
| `client-node-modules`     | Frontend-এর `node_modules`                              |

```powershell
docker volume create ph-pg-data
docker volume create server-node-modules
docker volume create server-logs
docker volume create client-node-modules
```

> **কেন `node_modules` আলাদা volume-এ রাখা হয়?**
> সোর্স কোড bind-mount করা হয় (`-v "${PWD}/backend:/app"`) যাতে লাইভ কোড এডিট সাথে সাথে দেখা যায়, কিন্তু `node_modules` Windows আর Linux (Alpine) কন্টেইনারের মধ্যে বাইনারি ইনকম্প্যাটিবিলিটি তৈরি করতে পারে। তাই সেটাকে আলাদা named volume-এ isolate করে রাখা হয়।

---

## 3. PostgreSQL Database কন্টেইনার

```powershell
docker run -d --name ph-db `
  --network ph-net `
  -e POSTGRES_HOST_AUTH_METHOD=trust `
  -e POSTGRES_DB=ph-health `
  -v ph-pg-data:/var/lib/postgresql/data `
  postgres:16-alpine
```

> ⚠️ `POSTGRES_HOST_AUTH_METHOD=trust` শুধু **লোকাল ডেভেলপমেন্টের জন্য**। এটা পাসওয়ার্ড ছাড়াই কানেকশন allow করে দেয়, তাই প্রোডাকশনে কখনো ব্যবহার করবেন না।

যাচাই করুন:

```powershell
docker ps
docker logs ph-db
```

লগে `database system is ready to accept connections` দেখলে DB প্রস্তুত।

---

## 4. Backend Dockerfile

`backend/Dockerfile`:

```dockerfile
FROM node:22-alpine

WORKDIR /app

RUN corepack enable && corepack prepare pnpm@10.20.0 --activate

COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .

EXPOSE 5000

CMD ["sh", "-lc", "pnpm generate && pnpm dev"]
```

বিল্ড:

```powershell
docker build -t ph-server-dev ./backend
```

---

## 5. Backend `.env` ফাইল

`backend/.env`-এ **কোনো ভ্যালুর চারপাশে ডাবল-কোট (`"`) ব্যবহার করবেন না** — `docker run --env-file` কোট চিহ্নকে literal ক্যারেক্টার হিসেবে ধরে নেয়, যেটা `DATABASE_URL` parse করার সময় ভাঙে।

```dotenv
NODE_ENV=development
PORT=5000

DATABASE_URL=postgresql://postgres@ph-db:5432/ph-health?schema=public

BETTER_AUTH_SECRET=your-secret-key-here
BETTER_AUTH_URL=http://localhost:5000

ACCESS_TOKEN_SECRET=accesssecret
REFRESH_TOKEN_SECRET=refreshsecret
ACCESS_TOKEN_EXPIRES_IN=1d
REFRESH_TOKEN_EXPIRES_IN=7d

FRONTEND_URL=http://localhost:3000
```

> hostname হিসেবে `ph-db` কাজ করছে কারণ `ph-server` ও `ph-db` দুটোই `ph-net` নেটওয়ার্কে আছে।

---

## 6. Backend কন্টেইনার রান করা

```powershell
docker run -d --name ph-server `
  --network ph-net `
  --env-file ./backend/.env `
  -e CHOKIDAR_USEPOLLING=1 `
  -e CHOKIDAR_INTERVAL=300 `
  -p 5000:5000 `
  -v "${PWD}/backend:/app" `
  -v server-node-modules:/app/node_modules `
  -v server-logs:/app/logs `
  -w /app `
  ph-server-dev `
  sh -lc "CI=true pnpm install && pnpm generate && pnpm dev"
```

> **`CHOKIDAR_USEPOLLING` / `CHOKIDAR_INTERVAL` কেন লাগে?**
> Docker Desktop-এ bind mount ব্যবহার করলে native filesystem-এর `inotify` ইভেন্ট সবসময় কন্টেইনার পর্যন্ত পৌঁছায় না। Polling মোডে চালালে হট-রিলোড নিশ্চিতভাবে কাজ করে।

লাইভ লগ দেখতে:

```powershell
docker logs -f ph-server
```

---

## 7. Prisma Migration চালানো

প্রথমবার (migration ফাইল না থাকলে):

```powershell
docker exec -it ph-server sh -lc "pnpm exec prisma migrate dev --name init"
```

migration ফাইল আগে থেকে থাকলে:

```powershell
docker exec -it ph-server sh -lc "pnpm exec prisma migrate deploy"
```

Status চেক:

```powershell
docker exec -it ph-server sh -lc "pnpm exec prisma migrate status"
```

Migration শেষে রিস্টার্ট করুন (seed script আবার রান হওয়ার জন্য):

```powershell
docker restart ph-server
docker logs -f ph-server
```

---

## 8. Frontend Dockerfile (Next.js + Bun)

এই প্রজেক্টের `frontend/package.json`-এর `dev` script Bun ব্যবহার করে চালানো হয় (`bun --bun next dev --webpack`), তাই Dockerfile-এ Bun ইনস্টল করা **আবশ্যক** — শুধু Node/pnpm দিয়ে হবে না।

`frontend/Dockerfile`:

```dockerfile
FROM node:22

WORKDIR /app

# pnpm enable
RUN corepack enable
RUN corepack prepare pnpm@10.20.0 --activate

# Bun install (package.json এর dev script bun ব্যবহার করে বলে দরকার)
RUN npm install -g bun

COPY package.json pnpm-lock.yaml ./
RUN pnpm install

COPY . .

EXPOSE 3000

CMD ["pnpm", "dev"]
```

বিল্ড:

```powershell
docker build -t ph-client-dev ./frontend
```

> নোট: এখানে `node:22` (non-alpine) ব্যবহার করা হয়েছে, `node:22-alpine` না — কারণ Bun-এর কিছু নেটিভ বাইনারি Alpine-এর musl libc-তে ঝামেলা করতে পারে। যদি ইমেজ সাইজ কমাতে চান পরে `node:22-slim` চেষ্টা করে দেখতে পারেন, তবে dev stage-এ `node:22` সবচেয়ে নিরাপদ।

---

## 9. Frontend `.env` ফাইল

`frontend/.env`-এ (কোট ছাড়া, আগের নিয়ম অনুযায়ী):

```dotenv
NEXT_PUBLIC_BACKEND_URL=http://localhost:5000
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

আপনার প্রজেক্টের প্রয়োজন অনুযায়ী আরও env variable এখানে যোগ হবে।

---

## 10. Frontend কন্টেইনার রান করা

```powershell
docker run -d --name ph-client `
  --network ph-net `
  --env-file ./frontend/.env `
  -e CHOKIDAR_USEPOLLING=1 `
  -e CHOKIDAR_INTERVAL=300 `
  -p 3000:3000 `
  -v "${PWD}/frontend:/app" `
  -v client-node-modules:/app/node_modules `
  -w /app `
  ph-client-dev `
  sh -lc "CI=true pnpm install && pnpm dev"
```

লগ দেখুন:

```powershell
docker logs -f ph-client
```

ব্রাউজারে খুলুন: **[http://localhost:3000](http://localhost:3000)**

---

## 11. ⚠️ গুরুত্বপূর্ণ ফিক্স — `ERR_CONNECTION_REFUSED` (localhost:3000 খুললে কানেক্ট হয় না)

Container-এর ভিতরে Next.js log-এ `Ready` দেখালেও ব্রাউজারে `localhost:3000` refuse করতে পারে, কারণ dev server default-এ শুধু কন্টেইনারের internal `localhost`-এ bind করে, host machine থেকে reach করা যায় না।

**সমাধান:** `frontend/package.json`-এর `dev` script-এ `-H 0.0.0.0` যোগ করুন:

```jsonc
// আগে:
"dev": "bun --bun next dev --webpack"

// পরে:
"dev": "bun --bun next dev --webpack -H 0.0.0.0"
```

তারপর কন্টেইনার recreate করুন:

```powershell
docker rm -f ph-client

docker run -d --name ph-client `
  --network ph-net `
  --env-file ./frontend/.env `
  -e CHOKIDAR_USEPOLLING=1 `
  -e CHOKIDAR_INTERVAL=300 `
  -p 3000:3000 `
  -v "${PWD}/frontend:/app" `
  -v client-node-modules:/app/node_modules `
  -w /app `
  ph-client-dev `
  sh -lc "CI=true pnpm install && pnpm dev"

docker logs -f ph-client
```

Expected log আউটপুট:

```
▲ Next.js 16.1.6
- Local:   http://localhost:3000
- Network: http://0.0.0.0:3000

✓ Ready
```

এখন `http://localhost:3000` কাজ করার কথা।

---

## 12. পোর্ট কনফ্লিক্ট — `port is already allocated` / `ports are not available`

এরর দেখতে এরকম:

```
docker: Error response from daemon: ports are not available:
exposing port TCP 0.0.0.0:3000 -> ... bind: Only one usage of each socket
address is normally permitted.
```

এর মানে হোস্ট মেশিনের 3000 পোর্ট **অন্য কোনো প্রোগ্রাম বা কন্টেইনার ইতিমধ্যে ব্যবহার করছে**।

### ধাপ ১: কে পোর্ট ধরে রেখেছে চেক করুন

```powershell
Get-NetTCPConnection -LocalPort 3000 -ErrorAction SilentlyContinue | Select-Object -ExpandProperty OwningProcess
```

অথবা Docker কন্টেইনার লিস্ট চেক করুন:

```powershell
docker ps -a
```

### ধাপ ২ক: যদি এটা একটা পুরনো Docker কন্টেইনার হয়

```powershell
docker stop ph-client
docker rm -f ph-client
```

### ধাপ ২খ: যদি এটা হোস্টে সরাসরি চলা কোনো প্রসেস হয় (যেমন লোকাল Next.js dev server)

নির্দিষ্ট PID বন্ধ করতে:

```powershell
Stop-Process -Id <PID> -Force
```

**One-liner — 3000 পোর্টে যা-ই চলুক, সব বন্ধ করে দিতে:**

```powershell
Get-NetTCPConnection -LocalPort 3000 -ErrorAction SilentlyContinue | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
```

### ধাপ ৩: অথবা, বন্ধ করতে না চাইলে অন্য হোস্ট-পোর্ট ম্যাপ করুন

```powershell
-p 3001:3000
```

তাহলে অ্যাক্সেস করবেন `http://localhost:3001` দিয়ে (কন্টেইনারের ভিতরে অ্যাপ এখনো 3000-এই চলবে)।

---

## 13. সব একসাথে চেক করা

```powershell
docker ps
```

৩টা কন্টেইনার `Up` স্টেটাসে থাকা উচিত: `ph-db`, `ph-server`, `ph-client`।

- Backend → [http://localhost:5000](http://localhost:5000)
- Frontend → [http://localhost:3000](http://localhost:3000)

---

## 14. কমন কমান্ড চিট-শিট

| কাজ                                       | কমান্ড                                                       |
|----------------------------------------------|-------------------------------------------------------------|
| সব কন্টেইনার দেখা (বন্ধ থাকলেও)             | `docker ps -a`                                               |
| কন্টেইনারের লাইভ লগ দেখা                    | `docker logs -f <container_name>`                            |
| কন্টেইনারের ভিতরে শেল ঢোকা                   | `docker exec -it <container_name> sh`                        |
| কন্টেইনার স্টপ করা                          | `docker stop <container_name>`                                |
| কন্টেইনার স্টপ + রিমুভ একসাথে                 | `docker rm -f <container_name>`                               |
| Image রিমুভ করা                             | `docker rmi <image_name>`                                     |
| ব্যবহৃত না-হওয়া রিসোর্স ক্লিন করা             | `docker system prune`                                         |
| ইমেজ/কন্টেইনারের ডিস্ক ইউসেজ দেখা              | `docker system df`                                             |
| ভলিউম রিমুভ করা                             | `docker volume rm <volume_name>`                               |
| নেটওয়ার্কে কারা যুক্ত আছে দেখা               | `docker network inspect ph-net`                                |
| নির্দিষ্ট পোর্টে কে চলছে বের করা (Windows)     | `Get-NetTCPConnection -LocalPort <port> \| Select-Object -ExpandProperty OwningProcess` |
| নির্দিষ্ট PID বন্ধ করা                         | `Stop-Process -Id <PID> -Force`                                |

---

## 15. সচরাচর সমস্যা ও সমাধান (Troubleshooting Index)

| এরর মেসেজ                                                      | কারণ                                                             | সমাধান |
|----------------------------------------------------------------------|-------------------------------------------------------------------|---------|
| `Environment variable DATABASE_URL is required but not set`          | `.env` ফাইল খালি/ভুল লোকেশনে                                     | `.env`-এ `DATABASE_URL` চেক করুন |
| `Can't reach database server at ph-db:5432`                          | `ph-db` কন্টেইনার বন্ধ, বা একই নেটওয়ার্কে নেই                     | `docker ps -a`, `--network ph-net` নিশ্চিত করুন |
| `The scheme is not recognized in database URL`                        | `.env`-এ ভ্যালুর চারপাশে quote (`"`) আছে                          | Quote সরান — `--env-file` কোট সাপোর্ট করে না |
| `Invalid base URL` / `Invalid URL`                                     | ভ্যালুর শেষে trailing space, বা ইনলাইন `#` কমেন্ট                  | Trailing space সরান, কমেন্ট আলাদা লাইনে নিন |
| `The table 'public.user' does not exist`                              | Migration রান হয়নি                                                | `prisma migrate dev --name init` চালান |
| `container is not running` (exec করার সময়)                          | কন্টেইনার ক্র্যাশ করে বন্ধ হয়ে গেছে                                | `docker logs <container_name>` দিয়ে কারণ বের করুন |
| `Conflict. The container name is already in use`                      | আগের একই নামের কন্টেইনার এখনো আছে                                  | `docker rm -f <container_name>` |
| `PowerShell: Missing expression after unary operator '--'`            | Bash-স্টাইল `\` লাইন-ব্রেক ব্যবহার করা হয়েছে                       | `\` এর বদলে PowerShell-এ `` ` `` (ব্যাকটিক) ব্যবহার করুন |
| `ports are not available` / `port is already allocated`               | হোস্টের পোর্ট আগে থেকেই অন্য কিছু ব্যবহার করছে                       | সেকশন ১২ দেখুন — পুরনো প্রসেস/কন্টেইনার বন্ধ করুন বা অন্য পোর্ট ম্যাপ করুন |
| `ERR_CONNECTION_REFUSED` (log-এ Ready দেখালেও ব্রাউজারে কাজ করছে না)   | Dev server শুধু কন্টেইনারের internal `localhost`-এ bind হয়েছে      | সেকশন ১১ দেখুন — `dev` script-এ `-H 0.0.0.0` যোগ করুন |

---

## 16. সবকিছু বন্ধ ও ক্লিন করা

```powershell
docker rm -f ph-server ph-client ph-db
```

ডাটাবেজসহ পুরোপুরি রিসেট করতে চাইলে (⚠️ সব ডেটা মুছে যাবে):

```powershell
docker volume rm ph-pg-data server-node-modules server-logs client-node-modules
```

---

## 17. পরবর্তী উন্নতি (ঐচ্ছিক)

এই সব `docker run` কমান্ড বারবার হাতে না লিখে একটা `docker-compose.yml` ফাইলে পুরো স্ট্যাক (db + backend + frontend + network + volumes) সংজ্ঞায়িত করলে ভবিষ্যতে শুধু:

```powershell
docker compose up -d
docker compose down
```

দিয়েই পুরো প্রজেক্ট চালু/বন্ধ করা যাবে। প্রয়োজন হলে বলুন — এই ফাইলের সব তথ্য ব্যবহার করে একটা `docker-compose.yml` বানিয়ে দিতে পারি।
