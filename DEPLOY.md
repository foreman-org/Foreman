# Hosting the demo

A judge should be able to open a link at any hour without your laptop being awake. This puts
Foreman on one small server for a few dollars a month.

**The split that keeps it cheap:** reading documents (Docling, PyTorch, OCR) is slow and happens
only when you add a manual, so it stays on your own machine. Answering a question is light, about a
second of CPU. The server therefore runs everything *except* Docling, and receives a library you
ingested at home. A 2 vCPU / 4 GB server is enough; no GPU.

## 1. A server

Any provider works: Hetzner CX22 (~€4/month), DigitalOcean, Azure, an old box in the office.
Ubuntu 24.04, then:

```bash
ssh root@SERVER
curl -fsSL https://get.docker.com | sh
git clone https://github.com/foreman-org/Foreman.git /opt/foreman && cd /opt/foreman
```

## 2. Settings

Create `/opt/foreman/.env` (compose reads it; it is git-ignored):

```bash
SITE_ADDRESS=foreman.example.com     # your domain pointed at the server, or
                                     # 203-0-113-5.sslip.io  (your IP with dashes: no domain needed)
POSTGRES_PASSWORD=change-me
NEO4J_PASSWORD=change-me-too
ANTHROPIC_API_KEY=sk-ant-...         # omit to run the demo in extractive mode, which costs nothing
ANTHROPIC_WORKSPACE_ID=              # only for organization-level keys
FOREMAN_RATE_LIMIT=30                # questions per hour per visitor; 0 removes the cap
```

If you use a domain, point an A record at the server first. Caddy then gets a real certificate
automatically, which the phone camera and QR scanning require.

## 3. Start it

```bash
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml logs -f caddy   # watch the certificate being issued
```

Open `https://SITE_ADDRESS`. The library will be empty until the next step.

## 4. Send it the library

On the machine where you ingested the manuals:

```bash
bash deploy/export_library.sh                 # writes foreman-library.tar.gz
scp foreman-library.tar.gz root@SERVER:/opt/foreman/
```

On the server:

```bash
bash deploy/restore_library.sh foreman-library.tar.gz
curl -s https://SITE_ADDRESS/api/stats        # documents, pages, chunks, points, nodes
```

The restore loads PostgreSQL and the page images, then rebuilds Qdrant and Neo4j from PostgreSQL.

## 5. QR labels

Open **Library → Machines & tags**, set *Address printed on the labels* to `https://SITE_ADDRESS`,
and download the labels. They now keep working regardless of what is running on your laptop.

## Updating

```bash
cd /opt/foreman && git pull
docker compose -f docker-compose.prod.yml up -d --build
```

## Notes

- **Uploading a document on the server will fail on purpose.** That image has no Docling, and the
  error says so. Ingest at home and re-run steps 4.
- **Cost control.** The key on a public site spends real money. Set a spend limit in the Anthropic
  Console, keep `FOREMAN_RATE_LIMIT`, or leave the key out entirely: extractive mode still shows
  cited procedures, the correction loop and the review queue, for nothing.
- **Backups.** `deploy/export_library.sh` also works as a backup; the server's own data lives in
  Docker volumes.
