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
  You can use the rest of the website without it.
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

## More documentation

- [Backend README](csp-backend/README.md): settings, tests, scripts and detailed docs
- [Frontend README](Untitled-Collab-Project-Frontend/README.md): project structure and scripts
