# Foreman

Foreman is a troubleshooting assistant for the plant floor. A technician scans a machine's QR tag and then types, says or photographs the fault. Foreman answers with the safety warnings first, then the steps. Every step cites the manual page, table or figure it came from, and one tap opens that page with the region outlined.

If a step does not match the machine in front of them, the technician says so. Foreman then searches again or hands the problem to the document's owner. It never fills a gap with a guess.

```
Technician: "Conveyor stopped, HMI shows E-42"

  WARNING  Lock out Q1 and open breaker F7 before opening panel B.
           The relay board stays live from the UPS.                       [§4.2.3, p. 47]
  1. Lock out main disconnect Q1 and open breaker F7.                     [§4.2.3, p. 47]
  2. Open panel B and locate relay K3 on the relay board.                 [Fig. 12, p. 47]
  3. Measure continuity across pins 13 and 14 of K3.                      [Fig. 12, p. 47]
  4. If the circuit is open, replace relay K3 with part 3RT2016-1BB42.    [§4.2.3, p. 47]
  ...
```

## Contents

- [Why it exists](#why-it-exists)
- [How a technician uses it](#how-a-technician-uses-it)
- [System architecture](#system-architecture)
- [How a question becomes a cited answer](#how-a-question-becomes-a-cited-answer)
- [The correction loop](#the-correction-loop)
- [Data model](#data-model)
- [Repository layout](#repository-layout)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [Loading your own manuals](#loading-your-own-manuals)
- [Scanning machine tags with a phone](#scanning-machine-tags-with-a-phone)
- [Modes](#modes)
- [Demo walkthrough](#demo-walkthrough)
- [API reference](#api-reference)
- [Tests](#tests)
- [Deploying](#deploying)
- [Troubleshooting](#troubleshooting)

## Why it exists

When a machine faults, the answer is usually in a manual somewhere: a 400-page PDF, a service bulletin that replaced part of it, a scanned addendum, or something only the engineer who owns the document knows. A general chatbot will give a confident answer that may be wrong. On a plant floor, a wrong torque value or relay number is a safety problem.

Foreman is built around three rules:

1. **Every step is grounded.** A step has to cite a retrieved passage, and a second check removes any claim the passage does not support.
2. **Trust is visible.** Each citation shows whether its page is verified (text layer), unverified (OCR, "check the original") or quarantined (too poor to cite until someone reviews it).
3. **Gaps go to a person.** When the documents do not cover what the technician sees, the question goes to the document owner. Their answer becomes a fix note that later answers can cite.

## How a technician uses it

1. Scan the QR tag on the machine, with Foreman's **Scan tag** button or the phone's camera app. Foreman opens on that machine.
2. Describe the fault by typing, speaking or taking a photo of the HMI screen or the part.
3. Read the answer: warnings first, then short numbered steps, each with a citation chip.
4. Tap a chip to see the source page with the paragraph, table row or figure outlined.
5. Ask a follow-up ("and what is the torque spec?"). The conversation and the pages it already relies on carry forward.
6. If a step does not match the machine, tap **Not what I see** on that step. Foreman rewrites the procedure from other evidence, or escalates to the document owner.

## System architecture

```mermaid
flowchart TB
    tech["Technician<br/>phone at the machine"]
    eng["Engineer / document owner<br/>desktop"]

    subgraph Web["frontend/ : Next.js on :3001"]
        ui["Troubleshoot · Library · Escalations · History<br/>browser only talks to Next.js, /api/* is proxied"]
    end

    subgraph API["backend/ : FastAPI on :8000"]
        main["main.py<br/>routes, rate limit"]
        subgraph Write["Adding a document"]
            ingest["ingest.py + schematic.py<br/>parse, verify pages, chunk"]
        end
        subgraph Read["Answering a question"]
            answer["answer.py<br/>understand, draft, prove,<br/>follow-ups, correction loop"]
            search["search.py<br/>hybrid search, graph expansion, rerank"]
        end
    end

    subgraph Stores["Data stores : docker compose"]
        pg[("PostgreSQL :5433<br/>system of record")]
        qd[("Qdrant :6333<br/>dense + BM25 + codes")]
        neo[("Neo4j :7687<br/>asset graph")]
        files[("backend/data/<br/>PDFs, page images, photos")]
    end

    subgraph Models["Models"]
        docling["Docling<br/>layout, tables, OCR<br/>local"]
        fastembed["FastEmbed<br/>embeddings, reranker<br/>local"]
        claude["Claude API<br/>optional"]
    end

    tech --> ui
    eng --> ui
    ui --> main
    main --> ingest
    main --> answer
    answer --> search

    ingest --> docling
    ingest -. "figures, schematics, OCR check" .-> claude
    answer -. "photo, draft, claim check" .-> claude
    search --> fastembed

    ingest -- "write" --> Stores
    search -- "query" --> qd
    search -- "query" --> neo
    answer -- "log answer" --> pg
```

### Components

| Layer | Technology | Role |
|---|---|---|
| Web app | **Next.js** (App Router) + Tailwind | Phone-first technician view and desktop tools. The browser only talks to Next.js, which proxies `/api` to FastAPI. |
| API | **FastAPI** | Ingest, search, answer, correction loop, escalations, review queue. |
| Parsing | **Docling** | Layout detection, OCR (RapidOCR) and table structure. Every element keeps its page and bounding box. PyMuPDF renders page images. |
| System of record | **PostgreSQL** | Documents, pages, chunks with page, box, extractor and confidence, machines and tags, answer logs, flags, fix notes. |
| Search index | **Qdrant** | Three vectors per chunk (dense, BM25 sparse, exact identifiers), fused in one query with reciprocal rank fusion. |
| Graph | **Neo4j** | Machine, document, procedure, fault code, component, part and terminal nodes, for multi-hop retrieval. |
| Embeddings and reranker | **FastEmbed** (`bge-small-en-v1.5`, `bm25`, `ms-marco-MiniLM-L-6-v2`) | Run locally, no API calls. The cross-encoder orders candidates and drops off-topic ones. |
| Vision-language model | **Claude** (`claude-opus-5` by default), optional | Reads photos, describes figures, reads wiring diagrams into a netlist, cross-checks OCR, drafts answers and checks each claim. |

PostgreSQL is the only store that has to be backed up. Qdrant and Neo4j are derived from it and can be rebuilt at any time with `python -m app.rebuild`.

### Two ways to run it

Reading documents is slow and heavy: Docling and PyTorch, a few seconds per page. Answering a question takes about a second of CPU. So Foreman is split into two builds.

```mermaid
flowchart LR
    subgraph Home["Your machine: full build"]
        pdfs["PDF manuals"] --> full["backend with Docling<br/>requirements.txt"]
        full --> devstores[("Postgres + files")]
        devstores --> export["deploy/export_library.sh"]
    end

    export -- "foreman-library.tar.gz" --> restore

    subgraph Server["Server: serving build (docker-compose.prod.yml)"]
        restore["deploy/restore_library.sh"] --> prodpg[("Postgres + files")]
        prodpg -- "app.rebuild" --> prodidx[("Qdrant + Neo4j")]
        caddy["Caddy<br/>HTTPS :443"] --> web["web (Next.js)"] --> api["api (no Docling)<br/>requirements-serve.txt"]
        api --> prodpg
        api --> prodidx
    end

    phone["Phones on the floor"] --> caddy
```

- **Development / ingest** (`docker-compose.yml`): only the three data stores run in Docker. The backend and frontend run on your machine. Everything works, including uploads.
- **Hosted / serving** (`docker-compose.prod.yml`): data stores, API, web app and Caddy for automatic HTTPS. The API image has no Docling, so it stays under a gigabyte and runs on a 2 vCPU / 4 GB server. You ingest at home and ship the library over. See [DEPLOY.md](DEPLOY.md).

## How a question becomes a cited answer

```mermaid
sequenceDiagram
    autonumber
    participant T as Technician
    participant A as answer.py
    participant C as Claude (optional)
    participant S as search.py
    participant Q as Qdrant
    participant N as Neo4j
    participant P as PostgreSQL

    T->>A: question, photo, machine tag, follow_up_to
    A->>P: load earlier turns of the conversation
    A->>C: understand: read photo, resolve follow-up, search terms, codes
    A->>S: search(query, machine, codes, pages carried from earlier turns)
    S->>Q: dense + BM25 + exact-code query, filtered to machine and citable pages
    S->>N: procedures linked to the codes, components, parts, terminals
    S->>S: cross-encoder rerank, add code/graph boosts, drop off-topic
    S-->>A: top evidence chunks
    A->>C: draft warnings and steps, each citing chunk IDs (with figure/table crops)
    A->>C: prove: check each claim against its cited passages
    A->>A: drop unsupported claims, set confidence
    A->>P: log question, evidence, scores, model, answer, removed claims
    A-->>T: warnings, steps, citations, confidence
```

Without an API key, steps 3, 8 and 9 are skipped. Foreman quotes the steps of the best-matching procedure word for word, so every step is its own source.

### The five stages

1. **Ingest** (`app/ingest.py`). Docling parses layout, lists, tables and figures, and runs OCR on scanned pages. Items become chunks under their section heading, each with page, bounding box, extractor and confidence.
   - **Long tables are also indexed row by row.** The answer to "what is fault 2310" is one row of a table that runs for pages, so each row is its own citation and the viewer outlines that row alone.
   - **Long manuals can be ingested a chapter at a time** by giving a page range, which keeps ingest to minutes instead of hours.
   - **Citations use the page number printed on the page.** The offset is read from the running headers, so a chip says "p. 377" when that is what the page says.
   - **Wiring diagrams are read, not just captioned** (`app/schematic.py`). A figure that looks like a circuit goes to the vision model a second time and comes back as a netlist: devices, terminals and the wires between them. The netlist becomes a `schematic` chunk that is searched and cited like any other, and its terminals become graph nodes. Wires the model cannot follow are listed as unreadable rather than guessed.
2. **Verify.** Every page gets a status:

   | Page | Status | Citable? |
   |---|---|---|
   | Has a text layer | `verified` | Yes |
   | Scanned, Docling OCR score of 0.85 or higher | `unverified` | Yes, with "check the original" |
   | Scanned, low score | retried with full-page OCR at 2x, then `quarantined` | No, until an owner approves it |
   | Vision check finds a misread number or code | `quarantined` | No, until an owner approves it |

   Owners approve or reject pages in the review queue, and the change takes effect in Qdrant immediately.
3. **Find** (`app/search.py`).
   - Qdrant runs dense, BM25 and exact-identifier retrieval in one fused query, filtered to the machine's documents and to citable pages.
   - Neo4j adds chunks from procedures linked to the fault codes, components, parts and terminals in the question. For a terminal, that includes the drawing of the far end of the wire, which is what answers "what does X4:7 land on".
   - A cross-encoder reranks the candidates. Exact code matches and graph links add to the score. Candidates that are off-topic and not linked by code or graph are dropped, so an unrelated question finds nothing rather than something.
4. **Answer** (`app/answer.py`). Structured output: warnings and steps, each with the chunk IDs it rests on. A follow-up continues the thread through `queries.parent_id`: the machine, the earlier turns and the pages they cited are carried into the next search and shown to the model again.
5. **Prove.** Each claim is checked against its cited passages and unsupported ones are removed. The answer's confidence is *verified source*, *unverified page* or *not in the documents*. If the check itself cannot run, the answer is not marked verified.

## The correction loop

```mermaid
flowchart TD
    step["Technician taps <b>Not what I see</b> on a step<br/>'There is no K3, the relay is K4'"]
    step --> research["Search again with what they see<br/>Neo4j: procedures involving both K3 and K4"]
    research --> found{"New evidence<br/>covers it?"}
    found -- yes --> revised["<b>Revised</b><br/>remaining steps rewritten from the new source,<br/>old steps dropped, new citations"]
    found -- no --> escalated["<b>Escalated</b><br/>procedure pauses, flag goes to the document owner<br/>with the note, photo and cited page"]
    escalated --> inbox["Owner reads it in <b>Escalations</b><br/>and writes a fix note"]
    inbox --> fixnote["Fix note saved to Postgres,<br/>indexed in Qdrant, linked in Neo4j"]
    fixnote --> next["Next time on this machine,<br/>the fix note is a cited source"]
```

## Data model

**PostgreSQL** (`app/db.py`) is the system of record:

| Table | Holds |
|---|---|
| `assets` | Machines: tag (e.g. `CV-12`), name, location |
| `documents` | Manuals and bulletins: title, version, owner, owner contact, page range, ingest status |
| `document_assets` | Which documents cover which machines |
| `pages` | One row per page: printed label, image, size, verification status and reason |
| `chunks` | Paragraphs, table rows, figures, schematics and fix notes, each with page, bbox, extractor, confidence |
| `queries` | Every answer: question, machine, mode, model, evidence and scores, final answer, removed claims, `parent_id` for threads |
| `flags` | "Not what I see" reports: step, note, photo, outcome, fix note |

**Neo4j** (`app/graph.py`) holds the relationships a keyword search cannot:

```
(Asset)-[:DOCUMENTED_BY]->(Document)-[:HAS_PROCEDURE]->(Procedure)-[:HAS_CHUNK]->(Chunk)
(Procedure)-[:ADDRESSES]->(FaultCode)
(Procedure)-[:INVOLVES]->(Component)
(Procedure)-[:USES_PART]->(Part)
(Procedure)-[:INVOLVES_TERMINAL]->(Terminal)
(Asset)-[:HAS_FIX_NOTE]->(Chunk:FixNote)-[:ADDRESSES|INVOLVES|USES_PART]->(...)
(Chunk:Schematic)-[:SHOWS]->(Terminal)-[:CONNECTS_TO {wire}]->(Terminal)
(Terminal)-[:ON_DEVICE]->(Component)
```

**Qdrant** (`app/index.py`) stores three vectors per chunk: `dense` (meaning), `bm25` (keywords) and `codes` (exact identifiers such as `E-42`, `K3`, `3RT2016-1BB42`). The payload carries machine IDs and page status, so filtering happens inside Qdrant.

Fault codes, components, part numbers and terminals are recognised by the patterns in `app/entities.py`.

## Repository layout

```
Foreman/
  README.md
  DEPLOY.md                    Hosting on one small server
  docker-compose.yml           Development: PostgreSQL, Qdrant, Neo4j
  docker-compose.prod.yml      Hosted: data stores + api + web + Caddy
  Caddyfile                    HTTPS reverse proxy for the hosted build
  deploy/
    export_library.sh          Pack Postgres + page images on the ingest machine
    restore_library.sh         Load them on the server and rebuild Qdrant and Neo4j
  manuals/
    manifest.json              Per-manual settings for bulk import (PDFs are git-ignored)
  backend/
    requirements.txt           Full build, includes Docling and PyTorch (CPU)
    requirements-serve.txt     Serving build, no Docling
    .env.example               Settings, copy to .env
    app/
      main.py                  FastAPI routes, rate limit, startup
      config.py                Environment settings
      db.py                    PostgreSQL schema and helpers
      ingest.py                Docling parsing, page verification, chunking
      schematic.py             Wiring diagram to netlist
      index.py                 Qdrant: three vectors, fused query, rerank
      graph.py                 Neo4j asset graph
      entities.py              Fault code / component / part / terminal patterns
      search.py                Hybrid search, graph expansion, reranking
      search_text.py           Tokenising and identifier detection
      answer.py                Understand, draft, prove, follow-ups, correction loop
      llm.py                   Claude client with structured output
      sample.py                Generates and ingests the demo library
      import_folder.py         Bulk import from a folder of PDFs
      rebuild.py               Rebuild Qdrant and Neo4j from PostgreSQL
    tests/                     pytest suite (API, follow-ups, model path, schematics)
  frontend/
    app/                       Routes: /ask, /library, /library/[id], /inbox, /log
    components/                Ask, Evidence, WhyThisAnswer, Library, Inbox, Log, Scanner, Nav
    lib/                       API client and hooks
  landing/                     Marketing site, a separate Vite project (see landing/README.md)
```

## Getting started

### Prerequisites

- Docker (for PostgreSQL, Qdrant and Neo4j)
- Python 3.11 or newer
- Node.js 20 or newer
- About 5 GB of disk for Python packages and models. Docling and PyTorch are large downloads.
- Optional: an Anthropic API key. Without one, Foreman runs in extractive mode.

### 1. Start the data stores

```bash
git clone https://github.com/foreman-org/Foreman.git
cd Foreman
docker compose up -d
```

This starts PostgreSQL on port 5433 (not 5432, so it can run alongside another local Postgres), Qdrant on 6333 and Neo4j on 7474 (browser) and 7687 (bolt).

### 2. Start the backend

```bash
cd backend
python -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env                 # optional: set ANTHROPIC_API_KEY
uvicorn app.main:app --port 8000
```

Check that it can reach all three stores:

```bash
curl -s http://localhost:8000/api/health
# {"ok": true, "services": {"postgres": true, "qdrant": true, "neo4j": true}, "llm": false, "model": null}
```

Interactive API docs are at http://localhost:8000/docs.

### 3. Start the frontend

```bash
cd ../frontend
npm install
npm run dev                          # http://localhost:3001
```

For a production build: `npm run build && npm start`.

### 4. Wait for the demo library

On first start the backend builds a demo library in the background:

- a 48-page CV-12 conveyor drive manual (the E-42 relay procedure, a torque table and a wiring figure on page 47)
- a 2023 service bulletin that moves one check from relay K3 to K4
- a pump manual
- a two-page scanned addendum with no text layer

Docling takes about 5 minutes on a laptop CPU, and the **Library** page shows progress. The first run also downloads the Docling, OCR, embedding and reranker models. The API can be used while this runs.

To reset the demo, run `python -m app.sample --reset` from `backend/`. It clears all three stores. To start with an empty library, set `FOREMAN_SEED=0`.

## Configuration

Settings are read from environment variables, or from `backend/.env` (real environment variables take precedence). See `backend/.env.example`.

| Variable | Default | Purpose |
|---|---|---|
| `ANTHROPIC_API_KEY` | empty | Turns on the model. Without it, extractive mode. |
| `ANTHROPIC_WORKSPACE_ID` | empty | Only for organization-level keys. |
| `FOREMAN_MODEL` | `claude-opus-5` | Claude model to use. |
| `FOREMAN_OFFLINE` | unset | `1` forces extractive mode even with a key. |
| `FOREMAN_SEED` | `1` | `0` skips loading the demo library on first start. |
| `FOREMAN_RATE_LIMIT` | `0` | Questions per hour per visitor. `0` means no limit. The hosted build defaults to 30. |
| `FOREMAN_ACCELERATOR` | `auto` | Where Docling runs: `auto`, `cpu`, `mps` (Apple Silicon), `cuda`. |
| `FOREMAN_TABLE_MODE` | `accurate` | `fast` ingests faster, with less exact table cells. |
| `FOREMAN_OCR_MIN_CONFIDENCE` | `0.85` | OCR score below which a scanned page is quarantined. |
| `FOREMAN_DATA_DIR` | `backend/data` | File store for PDFs, page images and photos. |
| `DATABASE_URL` | `postgresql://foreman:foreman@localhost:5433/foreman` | PostgreSQL connection. |
| `QDRANT_URL` | `http://localhost:6333` | Qdrant connection. |
| `NEO4J_URI` / `NEO4J_USER` / `NEO4J_PASSWORD` | `bolt://localhost:7687` / `neo4j` / `foreman-graph` | Neo4j connection. |
| `FOREMAN_API_URL` (frontend) | `http://localhost:8000` | Where Next.js proxies `/api`. |

## Loading your own manuals

You can upload one PDF at a time from the **Library** page. For more than a few, import a folder:

```bash
cd backend
python -m app.import_folder ../manuals --asset ACS580 --owner "Drives engineering"
python -m app.import_folder ../manuals --manifest ../manuals/manifest.json
python -m app.import_folder ../manuals --manifest ../manuals/manifest.json --dry-run
```

A manifest gives each manual its own machine, owner and page range:

```json
[
  {
    "file": "acs580_firmware.pdf",
    "title": "ACS580 firmware manual - fault tracing",
    "version": "ABB, as published",
    "owner": "Drives engineering",
    "contact": "ext. 4402",
    "asset": "ACS580",
    "asset_name": "ABB ACS580 drive",
    "location": "Line 3, MCC panel",
    "pages": "381-402"
  }
]
```

Page ranges matter. Docling reads a few seconds per page, so a 460-page manual takes hours while its fault-tracing chapter takes minutes. Files already imported are skipped, so you can re-run the command as the folder grows.

Manufacturers publish their manuals for download (ABB, Siemens, Rockwell, Danfoss, Grundfos, SKF and others). Keep them out of the repo: `manuals/*.pdf` and `backend/data/` are git-ignored. [manuals/README.md](manuals/README.md) shows how to get the ABB manual used in the demo, and some good questions to ask once it is loaded.

## Scanning machine tags with a phone

Each machine's QR label encodes `<address>/ask?asset=CV-12`. A technician can scan it with Foreman's **Scan tag** button or with the phone's own camera app, and both open Foreman on that machine.

Set the address under **Library > Machines & tags > Address printed on the labels**, then download each label. The address is stored per browser, so labels stay valid even if you print them from localhost.

**Browsers only allow the camera and microphone on HTTPS or localhost.** Over `http://<laptop-ip>:3001` the scanner says so and offers manual tag entry instead. Two ways to get HTTPS on your network:

```bash
# A temporary public HTTPS address (no account needed, changes on every restart)
cloudflared tunnel --url http://localhost:3001

# Or serve the app over HTTPS with a local certificate
npm run dev -- --experimental-https   # Android: accept the warning. iOS: trust the certificate first.
```

In-app scanning uses the browser's `BarcodeDetector` where available (Chrome, Edge) and falls back to jsQR (Safari, Firefox).

## Modes

### Technician and engineer views

Every answer is the same grounded answer. The switch above the question box only changes how much of the reasoning is shown.

| | Technician | Engineer |
|---|---|---|
| For | The person at the faulted machine | Reliability, controls and document owners |
| Shows | Warnings and action steps only | The same, plus what the evidence shows and where it is thin |
| Below the answer | Nothing | **Why this answer?**: exact identifier matches, graph links that brought evidence in, page verification, pages carried from earlier turns, top evidence with reranker scores |
| Needs a model | No | No |

The **Why this answer?** panel is built from retrieval data Foreman already logs for every answer. It shows how the evidence was chosen. It is not the model describing its own reasoning.

### With or without a model

| | No API key (extractive mode) | `ANTHROPIC_API_KEY` set |
|---|---|---|
| Answers | Steps quoted word for word from the best-matching procedure | Claude drafts steps from the evidence, including crops of figures and tables |
| Claim check | Not needed, every step is source text | Each claim checked against its cited passage, unsupported ones removed and listed |
| Photos | Stored with the question and the flag | Read for fault codes, labels and part numbers, then used in search |
| Scanned pages | Docling OCR, marked unverified or quarantined by score | Same, plus a vision check of the transcript against the page image |
| Figures | Caption and printed labels | Also described by the model, so they are searchable |
| Wiring diagrams | Caption and printed labels | Read into a netlist that becomes graph nodes |
| Correction loop | Revises when a procedure names what the technician reported | Rewrites the rest of the procedure from new evidence |

If a model call fails (network error, rate limit), that request falls back to extractive mode.

## Demo walkthrough

About three minutes, using the demo library.

1. **Troubleshoot > CV-12.** Ask "Conveyor stopped, HMI shows E-42". The lock-out warning comes first, then 7 steps citing §4.2.3, Fig. 12 and Table 4-3 on page 47. Tap **Fig. 12, p. 47** to open the page with the relay figure outlined.
2. **Ask a follow-up:** "and what is the torque spec?" The machine, earlier steps and their pages carry forward, so the answer is the torque table on the same page, not the whole procedure again. **New question** leaves the conversation.
3. **Not what I see** on step 2: "There is no K3, that slot says SPARE, the relay is K4." Neo4j finds the procedure that involves both K3 and K4 (service bulletin SB-2023-04, filed under another conveyor). Steps 2-4 are rewritten for relay K4, citing the bulletin.
4. **Not what I see** on the torque step: "Terminals are push-in spring type X9Q." Nothing covers it, so the procedure pauses and escalates to Reliability engineering, with a call button.
5. **Escalations.** The owner reads the note and the cited page and publishes a fix note. It is written to Postgres, indexed in Qdrant and linked in Neo4j.
6. **Troubleshoot** again with "E-42, relay has X9Q push-in terminals". The fix note is now a cited source.
7. **Ask** "What does terminal X4:7 carry?" The answer comes from the OCR'd addendum and shows **Unverified page: check the original**.
8. **Library > Review queue.** Addendum page 2 is too faded to read, so it is quarantined and cannot be cited until approved. **Machines & tags** prints the QR label that opens `/ask?asset=CV-12`.
9. **Engineer mode.** Flip the switch and ask the E-42 question again. The procedure is the same. Underneath, **Why this answer?** shows the identifier that matched, the graph links that pulled in the bulletin, each page's status, and the top evidence with scores.
10. **History.** Each answer shows what was retrieved: reranker score, exact code matches and which graph links brought each chunk in.
11. Ask something that is not covered: "recalibrate the laser height scanner". The answer is "Not in the documents".

## API reference

All routes are under `/api`. Interactive docs: http://localhost:8000/docs.

| Method | Path | Purpose |
|---|---|---|
| POST | `/api/ask` | Form: `asset`, `question`, optional `photo`, `follow_up_to`, `mode` (`technician` or `engineer`). Returns a cited answer. |
| POST | `/api/queries/{id}/flag` | Form: `step_index`, `note`, optional `photo`. Returns `revised` or `escalated`. |
| GET | `/api/queries`, `/api/queries/{id}` | Answer history |
| GET | `/api/flags?status=open` | Escalations inbox |
| POST | `/api/flags/{id}/fixnote` | JSON `{author, text}`. The fix note becomes a source. |
| GET / POST | `/api/documents` | List documents, or upload a PDF (multipart: `file`, `title`, `version`, `owner`, `owner_contact`, `assets`, `page_from`, `page_to`) |
| GET / DELETE | `/api/documents/{id}` | Pages, statuses and extracted regions, or delete |
| PUT | `/api/documents/{id}/assets` | Set which machines a document covers |
| GET | `/api/review` | Review queue: quarantined and unverified pages |
| POST | `/api/pages/{id}/review` | JSON `{action: "approve" or "reject", reviewer}` |
| GET | `/api/pages/{id}/image` | Rendered page image |
| GET / POST | `/api/assets` | Machines and QR tags |
| GET | `/api/assets/{tag}` | One machine, its documents and recent questions |
| GET | `/api/health` | Status of each store and whether the model is on |
| GET | `/api/stats` | Rows, points and nodes per store |

Example:

```bash
curl -s -X POST http://localhost:8000/api/ask \
  -F asset=CV-12 \
  -F "question=Conveyor stopped, HMI shows E-42"
```

## Tests

The tests need the Docker services running. They use their own database (`foreman_test`), Qdrant collection and Neo4j namespace, run in extractive mode, and use a shortened 9-page manual, so they never touch the demo data.

```bash
docker compose up -d
cd backend
pytest
```

## Deploying

To host Foreman so anyone can open a link without your laptop being on, see [DEPLOY.md](DEPLOY.md). In short:

1. Get a small Linux server (2 vCPU / 4 GB is enough, no GPU) and install Docker.
2. Create `.env` with `SITE_ADDRESS`, database passwords and optionally `ANTHROPIC_API_KEY`.
3. `docker compose -f docker-compose.prod.yml up -d --build`. Caddy gets an HTTPS certificate automatically.
4. On your own machine, `bash deploy/export_library.sh`, then copy the archive to the server and run `bash deploy/restore_library.sh foreman-library.tar.gz`.

A public API key spends real money. Set a spend limit in the Anthropic Console, keep `FOREMAN_RATE_LIMIT`, or leave the key out and run in extractive mode for free.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `/api/health` shows a store as `false` | That container is not running. Run `docker compose ps`, then `docker compose up -d`. |
| Backend cannot connect to Postgres | Postgres is on port **5433**, not 5432. Check `DATABASE_URL`. |
| Library stays empty after first start | Docling is still reading the demo library, or downloading models. Watch the backend log. |
| Uploading a PDF fails on the server | Intended. The serving image has no Docling. Ingest locally and restore (see [DEPLOY.md](DEPLOY.md)). |
| Scanner or voice input says the camera is unavailable | The page is on plain HTTP. Use localhost, a cloudflared tunnel, or `--experimental-https`. |
| Answers are quoted, not drafted | No API key, `FOREMAN_OFFLINE=1`, or the model call failed and fell back. `/api/health` shows `"llm"`. |
| Search results look stale after changing stores | Rebuild the index and graph from Postgres: `python -m app.rebuild`. |
