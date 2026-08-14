# n8n + PostgreSQL + ngrok

Automated n8n deployment with PostgreSQL and an ngrok tunnel via Docker.

---

## Requirements

- [Docker](https://docs.docker.com/get-docker/) (v20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2+)
- An [ngrok](https://ngrok.com/) account (the free plan works)

---

## Step 1 — Install Docker

### Windows

1. Download and install [Docker Desktop](https://www.docker.com/products/docker-desktop/).
2. Enable **WSL 2 backend** during installation.
3. Restart your computer.
4. Verify the installation:
   ```bash
   docker --version
   docker compose version
   ```

### Linux (Ubuntu/Debian)

```bash
# Install Docker
curl -fsSL https://get.docker.com | sudo sh

# Add your user to the docker group (so sudo is not required)
sudo usermod -aG docker $USER

# Re-login, or run:
newgrp docker

# Verify
docker --version
docker compose version
```

### macOS

1. Download [Docker Desktop for Mac](https://www.docker.com/products/docker-desktop/).
2. Install and start it.
3. Verify:
   ```bash
   docker --version
   docker compose version
   ```

---

## Step 2 — Set up ngrok

1. Sign up at [ngrok.com](https://ngrok.com/).
2. Open [Dashboard → Your Authtoken](https://dashboard.ngrok.com/get-started/your-authtoken) and copy the token.
3. Open [Dashboard → Domains](https://dashboard.ngrok.com/domains) and create a free static domain (e.g. `my-n8n.ngrok-free.app`).

---

## Step 3 — Clone the repository

```bash
git clone <YOUR_REPOSITORY_URL>
cd N8N
```

> **macOS tip:** Dotfiles (`.env`, `.gitignore`, …) are hidden in Finder by default.
> Press **Command + Shift + .** to show them.

If the repository is already on disk, just `cd` into the project folder.

---

## Step 4 — Configuration

Copy the example file and fill it in:

```bash
cp .env.example .env
```

Edit `.env` in any text editor:

```env
# PostgreSQL — choose a strong password
POSTGRES_DB=n8n
POSTGRES_USER=n8n
POSTGRES_PASSWORD=your_strong_password

# ngrok — paste your token and domain
NGROK_AUTHTOKEN=2abc...your_token
NGROK_DOMAIN=my-n8n.ngrok-free.app

# n8n — host must match NGROK_DOMAIN
N8N_HOST=my-n8n.ngrok-free.app
WEBHOOK_URL=https://my-n8n.ngrok-free.app/
GENERIC_TIMEZONE=Europe/Kyiv

# Credentials encryption key — generate with: openssl rand -hex 16
N8N_ENCRYPTION_KEY=your_generated_key
```

> **Important:** `.env` contains passwords and tokens — it will not be committed
> (listed in `.gitignore`).
>
> **Even more important:** store `N8N_ENCRYPTION_KEY` somewhere safe outside this
> project. Losing it means permanent loss of access to all stored credentials.

---

## Step 5 — Start

```bash
docker compose up -d
```

Wait for the images to download (first run may take a few minutes).

Check that all services are running:

```bash
docker compose ps
```

You should see four containers with status `Up`:

| Container           | Status       |
| ------------------- | ------------ |
| n8n-postgres        | Up (healthy) |
| n8n-app             | Up (healthy) |
| n8n-ngrok           | Up           |
| n8n-postgres-backup | Up (healthy) |

n8n does not start instantly — it runs database migrations first (up to a minute).
ngrok waits until n8n is `healthy`, so a short pause on startup is expected.

---

## Step 6 — Access n8n

- **Via ngrok (external):** `https://my-n8n.ngrok-free.app`
- **Locally:** `http://localhost:5678`
- **ngrok dashboard (request inspector):** `http://localhost:4040`

On first login, n8n will ask you to create an owner account.

---

## Useful commands

```bash
# Follow logs for all services
docker compose logs -f

# Follow logs for a specific service (name in compose, not the container)
docker compose logs -f n8n

# Stop all services
docker compose down

# Stop and remove data (volumes) — this deletes the database and workflows
docker compose down -v

# Restart
docker compose restart

# Update n8n to the pinned image version (or bump the tag first)
docker compose pull n8n
docker compose up -d n8n
```

---

## Project structure

```
N8N/
├── docker-compose.yml   # Service definitions
├── .env.example         # Environment variable template
├── .env                 # Your secrets (not in git)
├── .gitignore           # Git exclusions
├── backups/             # Automatic PostgreSQL dumps (not in git)
└── README.md            # This guide
```

---

## Data persistence

Data lives in Docker volumes (not in this folder) and survives container restarts.
The Compose project name is `n8n`, so the volumes on disk are:

| Volume              | Contents                             |
| ------------------- | ------------------------------------ |
| `n8n_postgres_data` | PostgreSQL database                  |
| `n8n_n8n_data`      | Workflows, credentials, n8n settings |

`backups/` is a folder in the project (bind-mounted). `.env` also lives in the project folder.

> To wipe all data, use `docker compose down -v`.

---

## Moving the project folder

You can move or rename this folder on the same machine. Volumes stay attached because Compose uses a fixed project name (`n8n`), not the directory name.

```bash
# 1. Stop the stack — do not use -v
docker compose down

# 2. Move the whole folder (including .env and backups/)

# 3. Start from the new path
cd /new/path
docker compose up -d
```

Do not move the folder while containers are running: the `backups/` bind-mount keeps the old absolute path until you recreate the stack.

---

## Encryption key

`N8N_ENCRYPTION_KEY` in `.env` is the key n8n uses to encrypt all stored
credentials (API keys, service passwords).

> **Critical:** keep a separate backup of this key outside this folder
> (e.g. in a password manager). Without it, neither a database dump nor the
> volume can restore access to stored credentials.

Set the key once and **never change it**.

---

## Automatic backups

The `postgres-backup` service creates a compressed database dump in `backups/`
**once a week** and prunes old copies (keeps 4 weekly backups).

| Folder            | Contents                            |
| ----------------- | ----------------------------------- |
| `backups/last/`   | Latest dump (`n8n-latest.sql.gz`)   |
| `backups/weekly/` | Weekly copies, retained for 4 weeks |

Schedule and retention can be changed via `.env`
(`BACKUP_SCHEDULE`, `BACKUP_KEEP_DAYS`, `BACKUP_KEEP_WEEKS`, `BACKUP_KEEP_MONTHS`).

```bash
# Run a backup immediately (do not wait for the schedule)
docker compose exec postgres-backup /backup.sh

# Verify the latest dump
gzip -t backups/last/n8n-latest.sql.gz
```

### Restore from a backup

```bash
# 1. Stop n8n so it does not write to the DB during restore
docker compose stop n8n

# 2. Load the dump
gunzip -c backups/last/n8n-latest.sql.gz | \
  docker compose exec -T postgres psql -U n8n -d n8n

# 3. Start n8n again
docker compose start n8n
```

> Restore only works with the same `N8N_ENCRYPTION_KEY` that was used when the
> dump was created.

---

## Troubleshooting

| Problem                            | Fix                                                                             |
| ---------------------------------- | ------------------------------------------------------------------------------- |
| ngrok does not start               | Check `NGROK_AUTHTOKEN` and `NGROK_DOMAIN` in `.env`                            |
| n8n cannot connect to the database | Check `POSTGRES_PASSWORD` — it must match for both services                     |
| Port 5678 is already in use        | Stop the other process (`docker ps`) or change the port in `docker-compose.yml` |
| `ERR_NGROK_8012` on the ngrok page | n8n is still starting — wait, or run `docker compose logs n8n`                  |
| Backups are not appearing          | `docker compose logs postgres-backup` — check password and schedule             |
| Webhooks do not work               | Ensure `WEBHOOK_URL` matches your ngrok domain                                  |
| Container keeps restarting         | `docker compose logs <service>` — inspect the logs                              |
