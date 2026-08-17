# Docker Compose শেখার গাইড (বাংলায়)

এই ফাইলটা তৈরি করা হয়েছে `docker-compose.yaml` ফাইলটাকে লাইন বাই লাইন বুঝিয়ে দেওয়ার জন্য। ভবিষ্যতে যখনই নতুন কোনো প্রজেক্টের জন্য `docker-compose.yaml` লিখবে, তখন এই ফাইলটা দেখে দেখে বানাতে পারবে।

---

## ১. Docker Compose আসলে কী?

একটা প্রজেক্টে সাধারণত একাধিক সার্ভিস থাকে — যেমন এই প্রজেক্টে আছে:
- একটা **Database** (Postgres)
- একটা **Backend server** (Node/Express জাতীয় কিছু, Prisma ব্যবহার করছে)
- একটা **Frontend client** (Next.js)

এই তিনটাকে আলাদা আলাদা `docker run` কমান্ড দিয়ে চালানো যায়, কিন্তু সেটা কষ্টকর ও ভুল হওয়ার সম্ভাবনা বেশি। **Docker Compose** একটাই YAML ফাইলে (`docker-compose.yaml`) সব সার্ভিস সংজ্ঞায়িত করতে দেয়, আর একটা কমান্ডে (`docker compose up`) সবগুলো একসাথে চালু করা যায়।

---

## ২. ফাইলের কাঠামো (Top-level Structure)

```yaml
services:   # কোন কোন container চলবে
networks:   # container গুলো একে অপরের সাথে কীভাবে কথা বলবে
volumes:    # কোন ডেটা container বন্ধ হলেও থেকে যাবে (persist করবে)
```

এই তিনটা হলো একটা `docker-compose.yaml` ফাইলের প্রধান তিনটা অংশ। আমাদের ফাইলেও ঠিক এই তিনটাই আছে।

---

## ৩. `services` — প্রতিটা Container এর সংজ্ঞা

### ৩.১ Database Service (`ph-db`)

```yaml
ph-db:
  image: postgres:16-alpine
  container_name: ph-db
  networks:
    - ph-net
  environment:
    - POSTGRES_USER=postgres
    - POSTGRES_PASSWORD=secret123
    - POSTGRES_DB=ph-helth
  volumes:
    - ph-pg-data:/var/lib/postgresql/data
  restart: always
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres -d ph-helth"]
    interval: 5s
    timeout: 5s
    retries: 10
    start_period: 10s
```

| কী | ব্যাখ্যা |
|---|---|
| `image: postgres:16-alpine` | নিজে Dockerfile না লিখে, Docker Hub থেকে রেডিমেড Postgres ইমেজ ব্যবহার করা হচ্ছে। `alpine` মানে হালকা ওজনের (ছোট সাইজ) ভার্সন। |
| `container_name: ph-db` | container-টার নাম fix করে দেওয়া হলো, না হলে Docker নিজে থেকে random নাম দেয়। |
| `networks: - ph-net` | এই container টা `ph-net` নামের নেটওয়ার্কে যুক্ত থাকবে (নিচে ব্যাখ্যা আছে)। |
| `environment` | Postgres ইমেজ ভেতরে ভেতরে এই env variable গুলো পড়ে ইউজার/পাসওয়ার্ড/ডাটাবেস অটো তৈরি করে নেয়। |
| `volumes` | `ph-pg-data` নামের একটা named volume, container-এর `/var/lib/postgresql/data` ফোল্ডারের সাথে যুক্ত। এটা না থাকলে container রিস্টার্ট/ডিলিট হলে সব ডেটা মুছে যাবে। |
| `restart: always` | container কোনো কারণে বন্ধ হয়ে গেলে Docker নিজে থেকেই আবার চালু করবে। |
| `healthcheck` | Docker কে জানানো হচ্ছে কীভাবে বুঝবে যে database "রেডি" হয়েছে কিনা — `pg_isready` কমান্ড চালিয়ে চেক করা হয়, প্রতি ৫ সেকেন্ডে, সর্বোচ্চ ১০ বার, শুরুতে ১০ সেকেন্ড সময় দিয়ে। |

**⚠️ গুরুত্বপূর্ণ নোট:** `POSTGRES_PASSWORD=secret123` — পাসওয়ার্ড সরাসরি ফাইলে লেখা আছে। শেখার জন্য ঠিক আছে, কিন্তু real project-এ এটা কখনো করবে না। এটা `.env` ফাইলে রাখা উচিত (নিচে ৫ নং সেকশনে দেখো)।

---

### ৩.২ Backend Service (`ph-server`)

```yaml
ph-server:
  container_name: ph-server
  build:
    context: ./backend
    dockerfile: Dockerfile
  restart: unless-stopped
  depends_on:
    ph-db:
      condition: service_healthy
  networks:
    - ph-net
  env_file:
    - ./backend/.env
  environment:
    CHOKIDAR_USEPOLLING: "1"
    CHOKIDAR_INTERVAL: "300"
  ports:
    - "5000:5000"
  working_dir: /app
  volumes:
    - ./backend:/app
    - server-node-modules:/app/node_modules
    - server-logs:/app/logs
  command: sh -lc "CI=true pnpm install && pnpm generate && pnpm prisma migrate deploy && pnpm dev"
```

| কী | ব্যাখ্যা |
|---|---|
| `build: context / dockerfile` | এই সার্ভিসের জন্য রেডিমেড image নেই, বরং `./backend` ফোল্ডারে থাকা `Dockerfile` দিয়ে নিজে ইমেজ বানাতে হবে। |
| `restart: unless-stopped` | কেউ ইচ্ছাকৃতভাবে বন্ধ না করলে, container ক্র্যাশ করলেও আবার চালু হবে। |
| `depends_on: ph-db: condition: service_healthy` | এই সার্ভিস তখনই চালু হবে যখন `ph-db` এর healthcheck "healthy" দেখাবে। শুধু `depends_on: [ph-db]` লিখলে শুধু "container চালু হয়েছে" চেক হতো, ডাটাবেস আসলেই রেডি কিনা তা নয় — এজন্য `condition` জরুরি। |
| `env_file: ./backend/.env` | `.env` ফাইলে থাকা সব variable এই container-এর ভেতরে env variable হিসেবে লোড হবে (যেমন DATABASE_URL, JWT_SECRET ইত্যাদি)। |
| `environment: CHOKIDAR_*` | Node-ভিত্তিক dev টুলগুলো (nodemon ইত্যাদি) ফাইল পরিবর্তন দেখার জন্য "polling" মোড ব্যবহার করবে — Docker-এর ভেতরে ফাইল-চেঞ্জ ডিটেকশন অনেক সময় ঠিকমতো কাজ করে না, তাই এই সেটিংস দরকার হয়। |
| `ports: "5000:5000"` | ফরম্যাট: `"হোস্ট-পোর্ট:কন্টেইনার-পোর্ট"`। মানে তোমার কম্পিউটারের ব্রাউজার/Postman থেকে `localhost:5000` এ গেলে, সেটা container-এর ভেতরের ৫০০০ পোর্টে যাবে। |
| `working_dir: /app` | container-এর ভেতরে কমান্ডগুলো `/app` ফোল্ডার থেকে চলবে। |
| `volumes: ./backend:/app` | তোমার লোকাল `backend` ফোল্ডারটা container-এর `/app` এর সাথে "mount" (সংযুক্ত) করা — মানে তুমি VSCode-এ কোড এডিট করলেই সাথে সাথে container-এর ভেতরেও পরিবর্তন দেখা যাবে, image আবার বানাতে হবে না। |
| `volumes: server-node-modules:/app/node_modules` | এটা একটা trick — `node_modules` কে আলাদা volume-এ রাখা হয়েছে যাতে হোস্ট মেশিনের (Windows/Mac) node_modules container-এর node_modules কে override না করে ফেলে (কারণ platform ভিন্ন হলে native module ক্র্যাশ করতে পারে)। |
| `command` | Container চালু হওয়ার পর এই শেল কমান্ডটা রান হবে: dependency install → prisma generate → migration apply → dev server চালু। |

---

### ৩.৩ Frontend Service (`ph-client`)

```yaml
ph-client:
  container_name: ph-client
  build:
    context: ./frontend
    dockerfile: Dockerfile
  restart: unless-stopped
  depends_on:
    ph-server:
      condition: service_started
  networks:
    - ph-net
  env_file:
    - ./frontend/.env
  environment:
    CHOKIDAR_USEPOLLING: "1"
    CHOKIDAR_INTERVAL: "300"
    WATCHPACK_POLLING: "true"
  ports:
    - "3000:3000"
  working_dir: /app
  volumes:
    - ./frontend:/app
    - client-node-modules:/app/node_modules
  command: sh -lc "CI=true pnpm install && pnpm exec next dev -H 0.0.0.0 -p 3000"
```

এখানে বেশিরভাগ কনসেপ্ট `ph-server` এর মতোই। যা নতুন:

| কী | ব্যাখ্যা |
|---|---|
| `depends_on: ph-server: condition: service_started` | এখানে `service_healthy` না লিখে `service_started` লেখা হয়েছে কারণ `ph-server`-এর কোনো `healthcheck` ডিফাইন করা নেই। শুধু container চালু হলেই যথেষ্ট ধরা হচ্ছে backend-এর জন্য অপেক্ষা করতে। |
| `WATCHPACK_POLLING` | Next.js নিজের ফাইল-ওয়াচার হিসেবে Webpack-এর Watchpack ব্যবহার করে, তাই এখানে আলাদা করে এই variable লাগে (Chokidar ভিত্তিক টুলগুলোর জন্য উপরের দুটোই যথেষ্ট)। |
| `-H 0.0.0.0` | Next.js dev server শুধু `localhost`-এ না শুনে, সব নেটওয়ার্ক ইন্টারফেসে শুনবে — এটা জরুরি, কারণ container-এর ভেতর থেকে "localhost" মানে container নিজেই, বাইরের হোস্ট মেশিন না। এটা না দিলে হোস্ট থেকে `localhost:3000` এ গিয়ে কানেক্ট করা যাবে না। |

---

## ৪. `networks` — Container রা একে অপরের সাথে কীভাবে কথা বলে

```yaml
networks:
  ph-net:
    driver: bridge
```

- `bridge` হলো সবচেয়ে কমন network driver — এটা একটা ভার্চুয়াল প্রাইভেট নেটওয়ার্ক বানায়, যেখানে একই নেটওয়ার্কে থাকা container গুলো একে অপরকে **সার্ভিস নাম দিয়ে** খুঁজে পায় (IP address দিয়ে না)।
- উদাহরণ: `ph-server`-এর `.env` ফাইলে DATABASE_URL এ হোস্ট হিসেবে `ph-db` লেখা থাকবে (IP address না), কারণ `ph-db` হলো container-এর নাম আর তারা একই `ph-net` নেটওয়ার্কে আছে। যেমন:
  ```
  DATABASE_URL="postgresql://postgres:secret123@ph-db:5432/ph-helth"
  ```

---

## ৫. `volumes` — ডেটা কীভাবে persist (সংরক্ষিত) থাকে

```yaml
volumes:
  ph-pg-data:
  server-node-modules:
  client-node-modules:
  server-logs:
```

এগুলো হলো **named volumes** — Docker নিজে এগুলো ম্যানেজ করে (তোমার প্রজেক্ট ফোল্ডারে দেখা যাবে না)। এখানে শুধু নাম ডিক্লেয়ার করা হয়েছে, উপরে সার্ভিসের ভেতরে এগুলো ব্যবহার হয়েছে।

**দুই ধরনের volume mount বোঝা জরুরি:**

1. **Named volume** (`ph-pg-data:/var/lib/postgresql/data`) — Docker ম্যানেজ করে, ডেটা persist থাকে, সরাসরি এডিট করা যায় না।
2. **Bind mount** (`./backend:/app`) — তোমার লোকাল ফোল্ডার সরাসরি container-এর সাথে সংযুক্ত, দুই দিকেই sync হয় (তুমি এডিট করলে container দেখবে, container এডিট করলে তুমি দেখবে)।

---

## ৬. একটা নতুন `docker-compose.yaml` লেখার Step-by-Step Checklist

ভবিষ্যতে নতুন প্রজেক্টে Compose ফাইল বানানোর সময় এই ধাপগুলো অনুসরণ করো:

### Step 1: কোন কোন সার্ভিস লাগবে তালিকা করো
> উদাহরণ: database, backend, frontend, redis, nginx ইত্যাদি।

### Step 2: প্রতিটা সার্ভিসের জন্য ঠিক করো — `image` না `build`?
- রেডিমেড software (Postgres, Redis, MongoDB) → `image:` ব্যবহার করো।
- নিজের কোড (backend/frontend) → `build: context / dockerfile` ব্যবহার করো।

### Step 3: `container_name` দাও (readability-র জন্য)

### Step 4: `networks` এ একটা কমন নেটওয়ার্ক যোগ করো
- একই প্রজেক্টের সব সার্ভিসকে একই নেটওয়ার্কে রাখো, যাতে তারা নাম দিয়ে একে অপরকে খুঁজে পায়।

### Step 5: `environment` / `env_file` ঠিক করো
- Secret / password / API key কখনো সরাসরি YAML-এ লিখো না — `.env` ফাইলে রাখো এবং সেটাকে `.gitignore`-এ যোগ করো।
- `env_file:` দিয়ে পুরো `.env` ফাইল লোড করা যায়, অথবা `environment:` দিয়ে আলাদা আলাদা variable সেট করা যায়।

### Step 6: `volumes` ঠিক করো
- Database/persist লাগবে এমন ডেটার জন্য named volume দাও।
- Dev মোডে কোড এডিট সাথে সাথে দেখতে চাইলে bind mount (`./folder:/app`) ব্যবহার করো।
- `node_modules`/`vendor` এর মতো dependency ফোল্ডারকে সবসময় আলাদা named volume এ রাখো, বাইরে থেকে override হওয়া থেকে বাঁচানোর জন্য।

### Step 7: `ports` ঠিক করো
- ফরম্যাট মনে রাখো: `"HOST:CONTAINER"`।
- একাধিক সার্ভিস একই host port ব্যবহার করলে conflict হবে — ইউনিক পোর্ট দাও।

### Step 8: `depends_on` দিয়ে চালু হওয়ার ক্রম ঠিক করো
- সাধারণ `depends_on: [service]` শুধু "container চালু হয়েছে" চেক করে।
- আসল "সার্ভিস রেডি কিনা" জানতে `condition: service_healthy` ব্যবহার করো, এবং সেই সার্ভিসে একটা `healthcheck:` অবশ্যই দিতে হবে।

### Step 9: `restart` policy ঠিক করো
- Database-এর মতো critical জিনিসে `restart: always`।
- App সার্ভিসে সাধারণত `restart: unless-stopped` যথেষ্ট।

### Step 10: `command` লেখার সময় সাবধান থাকো (এই প্রজেক্টে যে ভুলগুলো হয়েছিল)
- পুরো শেল কমান্ডটা **একটাই স্ট্রিং হিসেবে quote** করতে হয়:
  ```yaml
  # ❌ ভুল — এখানে "CI=true" আলাদা quote এ, বাকি কমান্ড container-এর বাইরে চলে যাবে
  command: sh -lc "CI=true" pnpm install && pnpm dev

  # ✅ সঠিক — পুরো কমান্ড একই quote এর ভেতরে
  command: sh -lc "CI=true pnpm install && pnpm dev"
  ```
- কমান্ড আর ফ্ল্যাগের নাম ভুল টাইপো আছে কিনা যাচাই করো (যেমন `exce` বনাম `exec`)।
- Dev সার্ভারকে `0.0.0.0`-এ বাইন্ড করো (`-H 0.0.0.0` বা `--host 0.0.0.0`), শুধু `localhost` না — না হলে হোস্ট মেশিন থেকে অ্যাক্সেস করা যাবে না।

### Step 11: টেস্ট করো
```bash
docker compose config      # ফাইল সিনট্যাক্স ঠিক আছে কিনা যাচাই করে
docker compose up --build  # সব সার্ভিস build করে চালু করে
docker compose ps          # কোন কোন container চলছে, health status কী
docker compose logs -f ph-server   # নির্দিষ্ট সার্ভিসের লগ দেখা
docker compose down        # সব container বন্ধ করা (volume থেকে যাবে)
docker compose down -v     # container + volume সব মুছে ফেলা (সাবধান, ডেটা হারাবে)
```

---

## ৭. কমন ভুল যা নতুনরা করে (মনে রাখার জন্য)

| ভুল | সমস্যা | সমাধান |
|---|---|---|
| Secret সরাসরি YAML-এ লেখা | GitHub-এ push হয়ে গেলে leak হয় | `.env` ফাইলে রাখো + `.gitignore` |
| `depends_on` এ শুধু service নাম | DB চালু হওয়ার আগেই backend চালু হয়ে ক্র্যাশ করতে পারে | `condition: service_healthy` + `healthcheck` |
| একই key দুইবার (যেমন `working_dir` দুইবার) | YAML এ শেষেরটাই কার্যকর হয়, আগেরটা silently ignore হয় | ডুপ্লিকেট key চেক করো |
| `command` এ ভুল quoting | কমান্ডের একটা অংশ container-এর বাইরে/host শেলে চলে যায় | পুরো কমান্ড একটাই quoted string রাখো |
| `node_modules` বাইন্ড মাউন্ট করা | Host আর container এর OS আলাদা হলে native module ক্র্যাশ করে | `node_modules` কে আলাদা named volume এ রাখো |
| Dev সার্ভার `localhost`-এ বাইন্ড করা | Container-এর বাইরে থেকে অ্যাক্সেস করা যায় না | `0.0.0.0`-এ বাইন্ড করো |

---

## ৮. এই প্রজেক্টের পুরো ফ্লো (Big Picture)

```
docker compose up
        │
        ▼
① ph-db চালু হয় (Postgres) ── healthcheck pass না হওয়া পর্যন্ত অপেক্ষা
        │  (service_healthy)
        ▼
② ph-server চালু হয় ── pnpm install → prisma generate → migrate → dev server
        │  (service_started)
        ▼
③ ph-client চালু হয় ── pnpm install → next dev server (0.0.0.0:3000)

সব container একই "ph-net" নেটওয়ার্কে থাকায় একে অপরকে নাম দিয়ে খুঁজে পায়
(যেমন backend থেকে "ph-db:5432" দিয়ে database এ কানেক্ট হওয়া যায়)
```

---

**পরবর্তী প্রজেক্টে নতুন `docker-compose.yaml` লেখার সময়, এই ফাইলের "৬ নং Step-by-Step Checklist" আর "৭ নং কমন ভুল" অংশটাই সবচেয়ে বেশি কাজে লাগবে — ওই দুটো অংশ বারবার দেখো।**
