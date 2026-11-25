# TORmax

TORmax collects Thai government IT procurement documents (TORs) from several
procurement websites and shows them in one searchable place. Vendors can
bookmark TORs, see how well their skills match, and get alerts. Job listers can
also post their own projects.

This repository is the umbrella repo. It holds the shared Docker setup and
pulls in the two app repositories as Git submodules:

| Folder | What it is | Runs on |
| --- | --- | --- |
| [`Untitled-Collab-Project-Frontend/`](https://github.com/Karczel/Untitled-Collab-Project-Frontend) | Website (Next.js) | http://localhost:3000 |
| [`csp-backend/`](https://github.com/GotAdithep/csp-backend) | API, scrapers and background jobs (Node.js + Express) | http://localhost:5001 |
| *(Docker only)* `mongo` | MongoDB database | localhost:27017 |

There are two ways to run the website on your computer:

- **[Option A: Docker](#option-a-run-everything-with-docker-recommended)** is
  recommended. One command starts the website, the API and the database.
- **[Option B: Without Docker](#option-b-run-without-docker)** runs each part
  with Node.js. Use it if you can't install Docker.

---

## 1. Get the code

Install [Git](https://git-scm.com/downloads), then clone **with submodules**:

```bash
git clone --recurse-submodules https://github.com/TORmax-KU/TORmax.git
cd TORmax
```

If you already cloned without `--recurse-submodules`, the two app folders are
empty. Fill them with:

```bash
git submodule update --init
```

> All commands below are run from this `TORmax` root folder unless a step says
> otherwise.

---

## Option A: Run everything with Docker (recommended)

### What you need

- [Docker Desktop](https://www.docker.com/products/docker-desktop/), installed
  and **running**. On Windows, use the default Linux containers (WSL 2) mode.

You do not need to install Node.js or MongoDB for this option.

### Steps

**1. Create the backend settings file.** This copies the example file, which
already works with Docker. Run it only once: it does not overwrite an existing
file.

PowerShell (Windows):

```powershell
if (!(Test-Path csp-backend/.env)) { Copy-Item csp-backend/.env.example csp-backend/.env }
```

macOS / Linux:

```bash
[ -f csp-backend/.env ] || cp csp-backend/.env.example csp-backend/.env
```

**2. (Recommended) Set a session secret** so you stay logged in when the
backend restarts. Generate a random value:

```bash
node -e "console.log(require('node:crypto').randomBytes(32).toString('hex'))"
```

If you don't have Node.js, any long random string works. Paste it into
`csp-backend/.env` after `SESSION_SECRET=`. If you leave it blank, the app
still works, but you are logged out every time the backend restarts.

**3. Start the stack.**

```bash
docker compose up -d --build
```

The first run downloads images and installs dependencies, which takes a few
minutes. Later starts are faster, and you can drop `--build` unless the code's
dependencies or Dockerfiles changed.

**4. Open the website** at **http://localhost:3000**.

The API is at http://localhost:5001. You can edit code in either app folder
while it runs: both apps reload automatically.

### Everyday Docker commands

| Task | Command |
| --- | --- |
| See what is running | `docker compose ps` |
| Follow the backend log | `docker compose logs -f backend` |
| Follow the website log | `docker compose logs -f frontend` |
| Stop everything (keeps your data) | `docker compose down` |
| Start again | `docker compose up -d` |
| After pulling changes that add npm packages | `docker compose up -d --build --renew-anon-volumes` |
| Delete the database and start fresh | `docker compose down -v` ⚠️ removes all local data |

---

## Option B: Run without Docker

### What you need

- [Node.js 22 or newer](https://nodejs.org/). Check with `node --version`.
- A MongoDB database. Pick one:
  - Install [MongoDB Community Server](https://www.mongodb.com/try/download/community)
    and leave it running on the default port 27017.
  - Run only the database in Docker:
    `docker run -d --name tormax-mongo -p 27017:27017 mongo:8.0`
  - Use a free [MongoDB Atlas](https://www.mongodb.com/atlas) cluster and use its
    connection string in step 2 below.

### Steps

You need **two terminals**: one for the backend and one for the website.

**Terminal 1: backend**

1. Install dependencies and create the settings file:

   ```powershell
   cd csp-backend
   npm ci
   if (!(Test-Path .env)) { Copy-Item .env.example .env }
   ```

   On macOS / Linux, use `[ -f .env ] || cp .env.example .env` for the last line.

2. Open `csp-backend/.env` and change the database address. The example
   points to the Docker database, which doesn't exist in this setup:

   ```dotenv
   MONGO_URI=mongodb://127.0.0.1:27017/fullstack_db
   ```

   If you use Atlas, paste your Atlas connection string here instead. You can
   also set `SESSION_SECRET` now, as described in Option A step 2.

3. Start the backend:

   ```bash
   npm run dev
   ```

   Wait until you see `Server running on port 5001`. Leave this terminal open.

**Terminal 2: website**

1. From the `TORmax` root folder:

   ```powershell
   cd Untitled-Collab-Project-Frontend
   npm ci
   if (!(Test-Path .env.local)) { Copy-Item .env.example .env.local }
   npm run dev
   ```

   On macOS / Linux, use `[ -f .env.local ] || cp .env.example .env.local`.

2. Open **http://localhost:3000**.

To stop either app, press `Ctrl + C` in its terminal.

---

## 2. Your first visit: why the feed is empty

A new database has **no government TORs**, so the home and search pages start
out empty. This is expected.

- The backend starts downloading listings from the government websites as soon
  as it connects to MongoDB. It repeats every five minutes while it is running.
- A government TOR only appears publicly after **AI analysis** (Google Vertex
  AI) has read its PDF and confirmed the details. That needs your own Google
  Cloud setup. See [csp-backend/docs/vertex-ai.md](csp-backend/docs/vertex-ai.md).
  You can use the rest of the website without it. The
  [public feed diagram](diagrams/8-activity-public-feed.png) lists every check
  a TOR must pass.
- Your database belongs to you. Pulling from Git does not copy a teammate's
  data.

To see the website with real content without setting up AI:

1. Go to http://localhost:3000/login and **register** with email and password.
   Choose the **Job Lister** role.
2. Create a project at http://localhost:3000/postings/new. Direct postings
   appear in search straight away.
3. Register a second account as a **Vendor** and fill in skills on the profile
   page. You can then try bookmarks, skill matching and notifications.

To use the **admin console** (`/admin`), turn an account into an admin. With
Docker:

```bash
docker compose exec -T backend node scripts/create-admin.js --email you@example.com
```

Without Docker, run `node scripts/create-admin.js --email you@example.com` inside
`csp-backend`. Sign out, sign in again, then open **Admin Portal** in the menu.

Google login and email alerts are optional. Each needs your own credentials.
See the [backend README](csp-backend/README.md#optional-features).

---

## Hosting (Vercel + Render)

This puts TORmax online for free. It uses three services:

| Part | Host | Free plan limits |
| --- | --- | --- |
| Database | [MongoDB Atlas](https://www.mongodb.com/atlas) | 512 MB of storage |
| Backend (`csp-backend`) | [Render](https://render.com) | 512 MB RAM. Sleeps after 15 minutes without requests (step 4 fixes this) |
| Website (`Untitled-Collab-Project-Frontend`) | [Vercel](https://vercel.com) | Hobby plan, non-commercial use only |

Vercel and Render deploy straight from each **app repository**. They don't use
this root repo or its submodules. Every push to an app repo's `main` branch
redeploys that app. To connect a repo, the host's GitHub app needs access to
it. The repo owner can grant that, or you can fork the repo and deploy your
fork.

Below, `<api>` stands for your Render address (e.g. `tormax-api.onrender.com`)
and `<site>` for your Vercel address (e.g. `tormax.vercel.app`).

### 1. Database: MongoDB Atlas

1. Create a free **M0** cluster and a database user.
2. Under **Network Access**, allow `0.0.0.0/0`, because Render's addresses
   change.
3. Copy the connection string and add a database name after the host, e.g.
   `mongodb+srv://user:password@cluster0.xxxxx.mongodb.net/tormax?retryWrites=true&w=majority`.

### 2. Backend: Render

1. **New → Web Service**, then connect the `csp-backend` repo.
2. Choose these settings:
   - **Language:** Node. Don't use Docker: the Dockerfile is for development.
   - **Build command:** `npm ci --omit=dev --ignore-scripts`
   - **Start command:** `npm start`
   - **Instance type:** Free
3. Add these environment variables. Leave `PORT` out, because Render sets it.

   | Variable | Value |
   | --- | --- |
   | `NODE_ENV` | `production` |
   | `MONGO_URI` | The Atlas connection string from step 1 |
   | `SESSION_SECRET` | A long random value (see Option A, step 2) |
   | `FRONTEND_URL` | `https://<site>` |

   Add any other settings you want from
   [csp-backend/.env.example](csp-backend/.env.example), such as the ingestion
   switches.
4. Deploy, then open `https://<api>/`. It should say `Backend is running`.

### 3. Website: Vercel

1. **Add New → Project**, then import the `Untitled-Collab-Project-Frontend`
   repo. Vercel detects Next.js by itself.
2. Add these environment variables, then deploy:

   | Variable | Value |
   | --- | --- |
   | `NEXT_PUBLIC_API_URL` | `/backend` |
   | `BACKEND_URL` | `https://<api>` |

   The website forwards `/backend/...` requests to Render, which keeps the
   login cookie on the website's own address. Calling Render directly from the
   browser would log people out right after they sign in.
3. If your Vercel address turns out different from the `FRONTEND_URL` you set
   on Render, update it there.

### 4. Keep the backend awake

Render's free backend sleeps after 15 minutes without requests. While it
sleeps, imports and matching stop, and the next visitor waits about a minute.
Create a free job on [cron-job.org](https://cron-job.org) that opens
`https://<api>/` every **10 minutes**. Render's 750 free hours a month cover one
service running all month.

### 5. Google login (optional)

In Google Cloud Console, add `https://<site>/backend/auth/google/callback` as an
authorized redirect URI. On Render, set `GOOGLE_CALLBACK_URL` to the same
address, and fill in `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET`.

### Good to know

- **Government TORs need AI analysis** to appear, as on your own computer (see
  [section 2](#2-your-first-visit-why-the-feed-is-empty)). For Vertex AI on
  Render, upload your service account JSON as a Render **Secret File** and set
  `GOOGLE_APPLICATION_CREDENTIALS=/etc/secrets/<file name>`.
- **Run only one importing backend per database.** If you point a local backend
  at the Atlas database, set `INGESTION_ENABLED=false` locally.
- **Make an admin** by running `node scripts/create-admin.js --email you@example.com`
  on your computer inside `csp-backend`, with `MONGO_URI` set to the Atlas
  string.
- **Logged out right after signing in?** Check `NODE_ENV=production` on Render
  and `NEXT_PUBLIC_API_URL=/backend` on Vercel. Redeploy Vercel after changing
  a `NEXT_PUBLIC_` variable, because its value is fixed at build time.

---

## Troubleshooting

| Problem | Fix |
| --- | --- |
| `Untitled-Collab-Project-Frontend` or `csp-backend` is empty | Run `git submodule update --init`. |
| `docker: command not found` or `Cannot connect to the Docker daemon` | Start Docker Desktop and wait until it says it's running. |
| `port is already allocated` / `EADDRINUSE` on 3000, 5001 or 27017 | Another program is using that port. This is often an Option B app or a local MongoDB still running while you start Docker, or the other way round. Stop it and try again. |
| Website loads but shows network errors | Make sure the backend is running and that http://localhost:5001 responds. Without Docker, check that `.env.local` has `NEXT_PUBLIC_API_URL=http://localhost:5001`. |
| Backend log shows `MongooseServerSelectionError` (Option B) | MongoDB isn't running, or `MONGO_URI` in `csp-backend/.env` still says `mongo` instead of `127.0.0.1`. |
| `Cannot find module …` after pulling changes | Option A: `docker compose up -d --build --renew-anon-volumes`. Option B: run `npm ci` again in the app folder. |
| Logged out every time the backend restarts | Set `SESSION_SECRET` in `csp-backend/.env`. |
| Backend log shows source timeouts or alerts | One government website is slow or down. That run is skipped and retried five minutes later. It doesn't affect the rest of the app. |

---

## Working with the submodules

Each app folder is its own Git repository. Commit and push changes **inside**
that folder, to that app's repository. Then commit in this root repo to record
which version of each app the root repo should use.

To update both apps to the latest commit on their default branches:

```bash
git submodule update --remote
```

## Diagrams

The [`diagrams/`](diagrams/) folder shows how TORmax works. Click a diagram's
name to expand it.

<details>
<summary><b>1. System architecture</b>: the website, API, background workers, MongoDB and outside services</summary>

![System architecture](diagrams/1-system-architecture.png)

</details>

<details>
<summary><b>2. ER diagram</b>: the database collections and how they relate</summary>

![ER diagram](diagrams/2-er-diagram.png)

</details>

<details>
<summary><b>3. Use cases</b>: what vendors, job listers and admins can do</summary>

![Use cases](diagrams/3-use-cases.png)

</details>

<details>
<summary><b>4. Ingestion and AI analysis</b>: how TORs are downloaded from government websites and analysed with Vertex AI</summary>

![Sequence diagram: ingestion and AI analysis](diagrams/4-seq-ingestion-ai.png)

</details>

<details>
<summary><b>5. Sign-in and sessions</b>: how logging in works</summary>

![Sequence diagram: sign-in and session](diagrams/5-seq-signin-session.png)

</details>

<details>
<summary><b>6. Matching and alerts</b>: how vendors are matched to TORs and notified</summary>

![Sequence diagram: matching and alerts](diagrams/6-seq-match-alerts.png)

</details>

<details>
<summary><b>7. Analysis status</b>: the states a TOR goes through during AI analysis</summary>

![State diagram: analysis status](diagrams/7-state-analysis-status.png)

</details>

<details>
<summary><b>8. Public feed</b>: the checks a TOR must pass before it appears publicly</summary>

![Activity diagram: public feed](diagrams/8-activity-public-feed.png)

</details>

## More documentation

- [Diagrams](#diagrams): architecture, database, use cases and main flows
- [Backend README](csp-backend/README.md): settings, tests, scripts and detailed docs
- [Frontend README](Untitled-Collab-Project-Frontend/README.md): project structure and scripts
