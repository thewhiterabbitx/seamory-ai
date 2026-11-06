<p align="center"><picture><source media="(prefers-color-scheme: dark)" srcset="frontend/public/seamory-logo-light.png"><img src="frontend/public/seamory-logo-dark.png" alt="Seamory AI" height="120"></picture></p>

# Seamory AI

**One governed description of your business data, and the knowledge around it, for people and for AI agents.**

Seamory AI has two halves that share one login, one settings page, one activity log, one MCP endpoint and one assistant:

- **The ontology**: a governed semantic model of your data. Entities, fields, relationships, the words people use ("open", "active", "unpaid") as exact conditions, and worked examples. It is versioned, reviewed and published like code, and it turns a question in plain language into correct SQL on your own database, under the same access rules every time.
- **The knowledge library**: what people *know about* the business and keep in documents, procedures and their heads. Files, web pages and interviews become cited, reviewable topic pages inside isolated **collections**, each with its own access. Topics can point at model entities (a policy *governs* Invoice), but the model never contains document content.

Both are served to AI agents through an MCP server and to people through a built-in assistant, **Alfred**.

It runs on your infrastructure: two containers to start (the app and PostgreSQL), plus optional ones you add when you need them. Your data stays in your database; Seamory AI reads it read-only, on demand.

This file is the guide to the whole platform: what it does, how to install it with or without the optional services, every page, every switch and setting, and how it works inside.

---

## Contents
1. [What you can do with it](#what-you-can-do-with-it)
2. [Why](#why)
3. [How it works](#how-it-works): [what it remembers](#what-it-remembers), [how a question is answered](#how-a-question-is-answered), [what we do differently](#what-we-do-differently)
4. [Install and first steps](#install-and-first-steps)
5. [Optional services](#optional-services)
6. [The app, page by page](#the-app-page-by-page)
7. [The ontology](#the-ontology): [model](#the-model), [drafts and publishing](#drafts-versions-and-publishing), [Excel](#the-excel-template), [search](#search), [demo database](#the-demo-database), [connecting your data](#connecting-your-data), [checking the model](#checking-the-model-against-the-data)
8. [The knowledge library](#the-knowledge-library)
9. [Agents: the MCP server](#agents-the-mcp-server)
10. [Alfred, the built-in assistant](#alfred-the-built-in-assistant)
11. [Access control and data protection](#access-control-and-data-protection)
12. [Administration](#administration): [users and roles](#users-roles-and-sign-in), [switches](#switches-and-flags), [observability](#observability), [settings reference](#settings-and-configuration-reference), [proxy](#running-behind-a-proxy), [operations](#operations)
13. [Security](#security)
14. [Architecture](#architecture)
15. [Development](#development)
16. [Limits](#limits)

---


## What you can do with it

| You want to... | Where | In short |
|---|---|---|
| **Describe your data once** | Ontology | Entities, fields (columns and calculated values), relationships, named filters, query examples, custom properties. Draft, review, publish, with an audit trail and optional four-eyes publishing. |
| **Bring your own spreadsheet or database** | Ontology > Changes & versions, Settings > Connectors | Fill in an Excel template, or point a connector (PostgreSQL, MySQL, MariaDB, ClickHouse, SQLite, DuckDB) at a database and generate a first draft from its structure. |
| **Let agents ask questions of your data safely** | MCP, Alfred | Plans for one correct SQL statement, run read-only under per-token or per-user entitlements; restricted columns never leave, personal ones are hashed. |
| **Turn documents into reliable, cited knowledge** | Knowledge | Upload files, capture web pages, or interview an expert. A model writes a verified source wiki for each document, a second model checks it claim by claim, and topics are written from the full wikis with quote-backed claims and typed relationships, all chunked, embedded and searchable. |
| **Keep knowledge separate where it must be** | Knowledge > Collections | Each collection is isolated with per-user and per-token grants and a *public* or *internal* usage tag that tells agents how to treat what they read. |
| **Review before anything is published** | Knowledge > Review | Contradictions, probable duplicates, pages the checker objected to, suggested model links, unreadable documents and vocabulary proposals wait in the collection's queue. |
| **Connect documents to the model** | Topic pages, entity pages, graph | Topics are linked to entities ("this policy governs Invoice"), shown on both sides, stored inside the collection and never in the model. |
| **Give any agent both** | MCP | Sixteen read-only tools: six for the model and the data, ten for documents (led by `library_context`, which returns whole topic pages with their graph; the document tools only appear for a token that can read a collection). The server explains itself, no prompt engineering. |
| **Ask in plain language** | Alfred | The in-app assistant uses the same tools and rules, in a side panel or full screen. |
| **Try it without real data** | Data > Browse | A synthetic database generated from your model, optionally with realistic values from the AI model. |
| **Run it your way** | Optional services | Separate worker, browser service for JavaScript pages, your own S3 bucket, any OpenAI-compatible AI endpoint, local embeddings. |
| **See everything that happened** | Admin > Activity | Every assistant conversation and agent session, step by step: model calls with tokens and cost, tool calls with timing and the response returned. |

## Why
Language models write SQL well, but they do not know your business: which table is the customer, which of three status columns means "open", that two tables must never be joined flat because sums double, that a column holds personal data, or that the refund policy changed last spring. Without that knowledge, an agent guesses, and its answers look right and are wrong.

Seamory AI is that knowledge, written down once, kept current by the people who own it, and enforced for every consumer:

- **One source of truth.** Definitions live in one published model and in governed collections, not in prompts, dashboards and people's heads.
- **Correct by construction.** Agents receive the tables, join keys and governed conditions for a question as one plan, and the database does the joining and aggregating.
- **Evidence for every fact.** Document knowledge is verified before it is used: each document becomes a checked source wiki, topics are written from those wikis, and every fact is backed by a claim with its exact quote, the source and the wiki version it came from. Anything doubtful waits for a person.
- **Safe by default.** What a caller may see is decided by the server, column by column and collection by collection, for every query and every search, whoever asks.
- **Any agent.** Anything that speaks MCP (Claude, ChatGPT-based tools, custom agents, IDE assistants) can use it with a token.


## How it works
Seamory AI keeps two memories, built differently on purpose and connected by a bridge. The **ontology** knows exactly what your data means and is used to query it. The **knowledge base** knows what your people know and can prove it.

### What it remembers

```mermaid
flowchart TB
  subgraph KB["The knowledge base: what your people know"]
    direction LR
    DOC["Documents<br/>files, web pages, interviews"] --> W["Source wiki<br/>verified page per document,<br/>every version kept"]
    W --> T["Topics<br/>pages by subject,<br/>written from the wikis"]
    W -. "quoted in" .-> C["Claims<br/>atomic facts with<br/>their exact quote"]
    C -- "prove" --> T
    T --- L["Relationships<br/>governs, part of, depends on"]
  end
  subgraph ONT["The ontology: what your data means"]
    direction LR
    M["Entities and fields<br/>definitions and filters<br/>relationships and joins<br/>sensitivity"] --> QT["Query tools<br/>plan one statement, check it,<br/>run it read-only"] --> DB[("Your databases<br/>read-only")]
  end
  KB -. "the bridge: a topic points at the entities it is about" .-> ONT
```

| Layer | What it is | Indexed as |
|---|---|---|
| **Original text** | The text of each document with its places (page, slide, sheet). | Passages: vectors and words |
| **Source wiki** | A standalone page written from one document, checked claim by claim against it; every version kept. | Every version chunked: vectors and words |
| **Topic** | A page by subject, merged across every document about it, following its type's template. | Every version chunked; the topic itself embedded with its claims |
| **Claim** | One atomic fact with its verbatim quote, the source, the wiki version and passage. Superseded, never deleted. | Embedded |
| **Relationship** | A typed link between two topics with why they relate. | Embedded with its context |
| **Ontology** | Entities, fields, definitions, filters, relationships, sensitivity, examples; versioned and published. | Search index of the published version |

Everything is searched by meaning and by words together (reciprocal rank fusion), always inside the collections the caller may read.

### How a question is answered

```mermaid
flowchart LR
  Q["A question<br/>from a person or an agent"]
  Q --> U["Understand<br/>definitions and joins"] --> P["Plan<br/>one statement"] --> X["Check and run<br/>rebuilt under your access,<br/>read-only"] --> A["An answer you can check<br/>numbers with their meaning,<br/>facts with their sources"]
  Q --> F["Find<br/>matching chunks of topic pages"] --> RD["Read<br/>whole pages and linked topics"] --> PR["Prove<br/>claims with quotes,<br/>source wiki if needed"] --> A
```

**Chunks find, pages answer.** For a document question the agent calls `library_context`: the question is matched against chunks of the published topic pages (and the topics' own vectors), the best chunk per topic is kept, results are reranked if a reranker is configured, and the agent receives the **whole page** with the matched passage, the graph of linked topics and the typed relationships that match, plus `answerable` (false means the documents do not cover it, and the agent is told not to answer from its own knowledge). Provenance (`library_topic_claims`, `library_topic_sources`), the graph (`library_traverse`) and the verified source wikis (`library_source_search`, `library_source`) are one call away. For a data question the agent calls `plan_query` and `query_data`: the plan carries the definitions and join paths, and the statement is parsed, checked against the caller's entitlements and rebuilt before it runs.

### What we do differently

| The usual approach | Seamory AI |
|---|---|
| Documents are chopped into passages and searched as they are. | Each document is first rewritten as a verified page, checked claim by claim against the original; anything doubtful waits for a person. |
| The assistant gets a handful of fragments and stitches an answer together. | The assistant gets whole topic pages by subject, already merged across every document that mentions the subject, plus the topics linked to them. |
| Sources are a list of file names at the end. | Every fact has a claim with its exact quote, the page version it came from and the passage it sits in. |
| When a document changes, old answers linger. | What a removed source supported is withdrawn and the affected pages are revised surgically, checked against the sources that remain. |
| Contradictions are averaged away. | A change that contradicts a published fact is held, both versions side by side, until someone decides. |
| Text-to-SQL guesses what "active" means. | The ontology defines it, and every statement is checked against the caller's access and rebuilt before it runs. |
| One big index for everyone. | Isolated collections with their own grants and a public or internal tag that travels with every result. |


## Install and first steps
Requirements: Docker with Compose v2, about 2 GB of memory for the app (the embedding model runs inside it), and an AI endpoint (an OpenRouter key, or your own OpenAI-compatible server) if you want Alfred and the Library's writing, checking and topic steps. Without one, everything else works: the model, search, agents, data, and the Library's storage and search (documents stay searchable by words and meaning; they just get no written page or topics until a model is configured).

```sh
cp docker-compose.example.yml docker-compose.yml   # adapt to your server if needed (proxy network, names, ports)
cp .env.example .env                               # fill in the REQUIRED values (openssl rand -hex 32 for secrets)
docker compose up -d
```
Open `http://localhost:8080` and sign in with `INITIAL_ADMIN_USERNAME` / `INITIAL_ADMIN_PASSWORD`. Then, in the order that usually makes sense:

1. **AI endpoint** (Settings > AI): address, key and model; press *Test the endpoint*. Optional, but Alfred and most of the Library need it.
2. **The ontology** starts empty and the home page offers two ways to start: on **Ontology > Changes & versions**, download the blank Excel template, fill it in, upload it and publish; or on **Settings > Connectors**, add your database, read its structure, generate a draft model, review it and publish.
3. **Try it**: build the demo database (**Data > Browse**) if you have no real data yet, ask **Alfred** a question, or create an MCP token (**Users & access > Tokens & MCP**) and connect an agent.
4. **Knowledge** (optional): create a collection (**Knowledge > Collections**), give people the *editor* role and a grant, and add documents on **Knowledge > Sources**.

| File | Purpose |
|---|---|
| `docker-compose.example.yml` | The two core services: `db` (PostgreSQL 16 with pgvector) and `app`, listening on `127.0.0.1:8080` by default. `./data` is mounted at `/app/data`. |
| `docker-compose.worker.yml`, `docker-compose.fetcher.yml`, `docker-compose.storage.yml`, `docker-compose.demo.yml` | The optional services, added with `-f` (see below). |
| `.env.example` | Every setting, marked required or optional, with the optional services' settings commented in. |
| `Dockerfile`, `Dockerfile.fetcher` | The app image and the browser service image. |
| `db/init/` | Runs once on a new database: creates the app's own non-superuser account, the vector extension and ownership. |
| `scripts/audit.sh` | Vulnerability scan of the locked Python packages and the frontend's runtime packages. |

## Optional services
The core is two containers. Everything else is an overlay file you add when you need it. Overlays combine: put every `-f` on the same command, and keep using that same command for `up`, `build` and `down`.

| Service | File | What it adds | You need it when... |
|---|---|---|---|
| **Worker** | `docker-compose.worker.yml` | A container that does the Library's background work (reading, writing pages, topics, link suggestions, interview drafts). The web app stops doing it (`SERVICE_ROLE=api`). | Many or large documents, several people uploading at once, or you want the web app to stay quick under load. Scale it with `--scale worker=3`. |
| **Browser service** | `docker-compose.fetcher.yml` | An isolated Chromium (Playwright) that renders web pages built by JavaScript. No database, no files, no app secrets; reaches the app over its own private network only. | You capture web addresses of single-page sites or pages that show their content only after scripts run. Plain pages (most articles and documentation) work without it. |
| **S3 bucket** | `docker-compose.storage.yml` | A self-hosted S3-compatible store ([SeaweedFS](https://github.com/seaweedfs/seaweedfs), Apache-2.0) plus a one-shot job that creates the bucket. | You want the Library's original files outside the app's data folder (several app hosts, backups at the storage layer, versioning, or simply object storage). Any other S3 service works without this file. |
| **Demo databases** | `docker-compose.demo.yml` | PostgreSQL, MariaDB and ClickHouse with the same synthetic data. | You want to try the connectors on realistic data. See [The demo database](#the-demo-database). |

### How to turn each one on
**1. Worker.** Nothing to configure: the file sets `SERVICE_ROLE=api` on the app and starts the worker with the same `.env` and the same `./data`.
```sh
docker compose -f docker-compose.yml -f docker-compose.worker.yml up -d --build
```

**2. Browser service.** Put in `.env` (or on **Settings > Web capture**):
```sh
FETCHER_TOKEN=<openssl rand -hex 32>     # the shared secret between the app and the fetcher
FETCHER_URL=http://fetcher:8001
FETCH_RENDER=auto                        # auto: render only pages with almost no text without JavaScript; always; never
```
```sh
docker compose -f docker-compose.yml -f docker-compose.fetcher.yml up -d --build
```
The app attaches to a private `fetch` network shared only with the fetcher (and the worker). The fetcher container is read-only, drops every capability, has no new privileges and has memory and process limits. Its browser never reaches the network itself: every request it makes is answered by the same address guard the app uses, so private, loopback and metadata addresses stay out of reach. To read an intranet, list the hosts in `FETCH_ALLOWED_HOSTS` (in `.env`, for the fetcher) **and** in Settings > Web capture (for the app).

**3. S3 bucket.** Pick an access key and a long random secret, then put in `.env` (or on **Settings > Storage**):
```sh
STORAGE_DRIVER=s3
S3_ENDPOINT=http://s3:8333
S3_BUCKET=library
S3_ACCESS_KEY=<pick one>
S3_SECRET_KEY=<pick one, long and random>
S3_PATH_STYLE=true
S3_REGION=us-east-1
```
```sh
docker compose -f docker-compose.yml -f docker-compose.storage.yml up -d
```
The `s3-init` job creates the bucket on the first start and exits. Press **Test the storage** on Settings > Storage to check the values. Nothing is published on the host: only the app (and the worker) reach the store. Choose the storage **before the first upload**: files already stored stay where they were and are not moved when the driver changes. Back up the `s3_data` volume (or use your own S3 service's tools). To use AWS S3, Ceph, Cloudflare R2 or a MinIO you run yourself, skip the file and fill in the same settings with your service's endpoint.

**4. Everything together**
```sh
docker compose -f docker-compose.yml -f docker-compose.worker.yml -f docker-compose.fetcher.yml -f docker-compose.storage.yml up -d --build
```

**Without any of them** the app does everything itself: workers inside the web process (`LIBRARY_WORKERS`, default 2), files in `./data/library`, web pages fetched as plain pages.

### Other things you can plug in (settings, not containers)
| What | Where | Notes |
|---|---|---|
| **AI endpoint** | Settings > AI | Any OpenAI-compatible server: OpenRouter, a LiteLLM gateway, Ollama, vLLM. Nothing here is tied to one vendor. |
| **A model per Library job** | Settings > AI | Writer, checker, analyzer, topic writer, interviewer and image reader can each use a different model; empty uses the assistant model. |
| **Embeddings** | `.env` | `local` (default, in the container, no key) or a remote model. Any vector size pgvector stores. |
| **Reranker** | Settings > AI | Optional Cohere-style `/rerank` endpoint that sharpens Library search; off when empty. |
| **Proxy** | `.env`, see [Running behind a proxy](#running-behind-a-proxy) | nginx, Traefik, a load balancer or Cloudflare in front, with the real client address read safely. |


## The app, page by page
The top bar has the **search box** (type to find entities, fields, query examples, collections, documents and topics, or to jump to any page; **Ctrl+K** puts the cursor there; results open in place and follow your own access), a **theme toggle** (light or dark, remembered in the browser; the system theme until you choose), your account chip (click to change your password) and sign out. **✦ Ask Alfred** floats at the bottom right. The side menu has four groups and a small tab on its edge that folds it to icons.

### Ontology
| Page | What it shows and does |
|---|---|
| **Entities** | Every entity grouped by domain, with a filter. Modellers create entities here (**New entity**) when a draft exists. A small *Ontology version* selector at the top right switches between the published version and the draft (the choice is remembered in the browser). |
| Entity page | Tabs: Fields, Calculated, Filters, Checks, Relationships, Examples, Example questions, plus **Related documents** (topics from the collections you can read that are about this entity). Every field shows meaning, type, role, sensitivity, agent access, allowed values and review status. In a draft, modellers edit everything in place. |
| **Graph** | The entity graph (solid = confirmed, amber = optional, red = unreliable). Tick *Show the topics of my document collections* to overlay them. Drag to pan, scroll to zoom. |
| **Join paths** | Pick two entities: the preferred path with each join and its reliability, the all-confirmed alternative, approximate paths through a shared parent (fan-out) and their cost. |
| **Query examples** | All query examples, filterable; modellers add and edit them in a draft; *Try on data* runs one. |
| **Changes & versions** | The draft (label, edits, what it is based on), **Start a draft**, **Validate**, **Publish** (with progress), **Discard**; all versions; the difference between the draft and the published version; the audit log; the Excel card (blank template, export, upload, backups). |

### Knowledge (the Library; can be turned off in Settings > Library)
| Page | What it shows and does |
|---|---|
| **Collections** | Your collections as cards (name, usage tag, your level, counts), with **Settings**, **Export** and **Delete** for managers, and the access table (who has read, write or manage, users and tokens). Admins see collections they hold nothing on and can grant themselves access, which is logged. |
| **Sources** | Upload files (drop them or browse), **Add a web page**, search inside the collection, and the list of sources with status and progress. A source opens to its **page**: the written, checked version of the document, its versions and checks, and the original download. |
| **Topics** | The collection's topics with type and filter; a topic page shows the article with citation chips, an evidence panel (every statement with its quote, source, place and original passage), aliases, related topics, links to the data model, version history, and **Merge into another topic**. |
| **Graph** | The topics of one collection and their links (colour = type), optionally with the model entities they are about. |
| **Review** | The collection's queue: what waits for a person to decide. The sidebar badge counts it. |
| **Interview** | Capture an expert's knowledge by conversation; see [Interviews](#interviews). |
| **Vocabulary** | The collection's kinds of topic and relation, what the analyzer proposed, and the manager's tools to add, edit, merge and delete. |
| **Health** | A check without any AI model: what needs attention in the collection. |

### Data
| Page | What it shows and does |
|---|---|
| **Browse** | Which database is in use (the demo database or a connector), its tables with row counts, grouped by schema. On the demo database, admins build or rebuild it here (scale, seed, AI step). |
| Table view | Rows of one table, paged (50 per page), sortable, with filters per column. Hover a header for the column's meaning. Keys open the related row. |
| Row view | One row with every field's business name, plus every table that points at it, with how many rows do. |
| **SQL console** | One read-only statement at a time (Ctrl+Enter runs), under your entitlements, up to `CONSOLE_MAX_ROWS` rows (or the connector's lower limit). |
| **Model check** | The model against the data, grouped, with filters per status. See [Checking the model](#checking-the-model-against-the-data). |

Everything on these pages follows your entitlements: masked columns show hashes, restricted columns are hidden unless granted.

### Admin
| Page | Who | What it shows and does |
|---|---|---|
| **Users & access** (non-admins see *Connect agents*) | everyone sees the MCP details; the rest is admins | Tabs: **Tokens & MCP** (endpoint, connection command, tokens with their entitlements) and **Users** (roles and entitlements). |
| **Activity** | admins | Every Alfred conversation and agent session, step by step, summaries per tool and per model, and Library jobs. |
| **Settings** | admins | Tabs: **General**, **AI**, **Ontology & data**, **Library**, **Web capture**, **Storage**, **Limits**, **Connectors** and **Maintenance** (search index, questions nobody could answer, empty the ontology). A search box finds a setting across all tabs, and each row shows whether its value comes from the page, `.env` or the default. |


## The ontology
The governed half: a model of your data that people edit and publish, and that agents use to plan queries. It holds **only ontology** (entities, fields, relationships, filters, checks and examples); what people know *about* the business lives in the [knowledge library](#the-knowledge-library).


## The model
Four concepts. Everything an agent or a person needs to know is one of them.

### Entities
The business things: a customer, an order, a contract, a sensor. Each entity has:

| Property | Meaning |
|---|---|
| key | Short `lower_snake_case` id used everywhere else (`customer`, `order_line`). |
| name, domain | Business name and business domain (Sales, Finance...). Domains group the entity list. |
| kind | `master`, `reference`, `transaction`, `fact`, `relationship`, `bridge`, `agreement`, `supertype`, `derived` or `technical`. Used by the planner and by the demo-data generator (how many rows). |
| definition | What one row of the entity is. |
| table | `schema.table` that holds it (`semantic_table`). |
| keys | Primary key; business key (what people use to identify it); system of record. |
| synonyms, not to confuse with | The words people actually use, and things it is often mistaken for. |
| belongs to, subtype of | A parent concept; the supertype entity, if any (a supertype table unifies several subtypes behind a discriminator). |
| model status | `confirmed`, `new` or `placeholder`. |
| standard | Optional: the matching concept in an industry standard you follow (resource, reference, detail). |
| lifecycle states | One line per status column: `status: draft > active > closed`. |
| example questions | Typical questions the entity answers (searchable). |

Entities also carry:
- **Filters**: named conditions such as *open order* = `closed_at IS NULL`, with synonyms, a description, a source and a status (`governed` = agreed, `working` = not confirmed yet). Agents get them pre-filled instead of guessing values.
- **Checks**: data-quality rules in words, optionally with SQL that counts violations (one column `n`, which must be 0). The Model check runs them.

### Fields
What an entity has:
- **Attributes** are columns of the entity's table.
- **Calculated fields** (role `calculated`) are values computed from the data, not columns: meaning, unit, and optionally the SQL that computes them. That SQL is context for agents; the app never runs it.

| Property | Meaning |
|---|---|
| column, business name | Column name (a short code for a calculated field) and the name people use. |
| role | `primary_key`, `foreign_key`, `attribute`, `alternate_reference`, `dimension`, `fact_measure`, `derived_attribute`, `lineage` or `calculated`. |
| type | `text`, `integer`, `decimal`, `boolean`, `date`, `timestamp`, `text[]`. |
| description | What the value means. |
| allowed values | The codes stored in the column, each with a label and a note. Agents get the exact values. |
| references | For a foreign key: the entity it points to. |
| format, example | A pattern ("two letters + 6 digits") and a sample value (never a restricted one). |
| business term | Glossary name and definition. |
| standard name, unit | Name in the industry standard; unit of the value (count, EUR, hours). |
| sensitivity | `internal`, `confidential`, `personal` or `restricted` (see [Access control](#access-control-and-data-protection)). |
| agent access | `full`, `masked` or `none`. Default: `none` for restricted, `full` otherwise. A restricted field must be `none`. |
| review status | How settled the definition is: `confirmed_by_team`, `by_design`, `high_confidence`, `needs_review` (default) or `source_needed`. Pending definitions are flagged to agents. |

### Relationships
How entities connect: from, predicate (the verb: *places*, *belongs to*), to, the exact join (`a.col = b.col`), cardinality (`many-to-one`, `one-to-many`, `one-to-one`, `many-to-many`, `many-to-zero-or-one (optional)`, `subtype-of`), and how reliable the link is:

| Status | Meaning |
|---|---|
| `confirmed` | The join holds. |
| `optional` | Holds when filled, may be empty. |
| `unreliable` | Often empty or wrong. Join planning avoids it when it can and says so when it cannot. |

Join planning prefers reliable links, offers an all-confirmed alternative when the best path is not, and warns when a path would match every sibling record through a shared parent (fan-out).

### Query examples
A worked query with its SQL, `:parameters`, a parameter → entity map and a sample query that returns one row of parameter values; the Model check runs each on real rows, and agents get the ones that apply to the entities of their question as patterns (up to three per query plan, since each carries SQL). Each has an id, a title (the question it answers), a note, optional topic and source, and the entities it applies to.

The model is **only** the structured ontology: entities, fields, relationships, filters, checks and examples. What people know *about* the business (rules, definitions, abbreviations, procedures, policies) is not kept here: it is ingested as ordinary documents into the Library, where grants, usage tags, evidence and review govern it, and tools bring it to an agent when it matters (`plan_query` lists the relevant topics, `describe` finds a name only the documents know, entity pages list related documents). Older Excel files with a Knowledge sheet still upload: their examples are read, the rest is reported and left out.

### Custom properties and model metadata
Any item (the model, entities, fields, relationships, filters, checks, examples) can carry properties of your own: an owner, a source system, a ticket number, a lineage note. They are kept as written, shown in the app, searchable, and given to agents. In Excel, any extra column is a custom property.

The model itself has a name, id, version, date, source, status text, the standards it follows, what each sensitivity class means in your organisation, a row-level rule in words, and reading notes.

SQL in the model (filters, checks, examples) is written once in PostgreSQL syntax and translated for whichever database is connected.

## Drafts, versions and publishing
There is at most one **draft** and one **published** version at a time; earlier published versions are **archived** (their content is kept, their search index is not).

1. **Start a draft**: a copy of the published version (or, with nothing published, a draft created by an Excel upload or a connector).
2. **Edit**: in the UI, by uploading Excel (replaces the draft when you choose to), or by generating from a connector. Every change is written to the audit log with who and when. Modellers and above edit.
3. **Validate**: references between items resolve, required properties are present, values are from the allowed lists, a restricted field has agent access `none`, filter names and check ids are unique per entity, examples have SQL, and the search chunks can be built. SQL in filters and checks that does not run on the connected database is reported as **advice**, never as a blocker: the model may be written before the database exists.
4. **Review**: compare the draft with the published version, item by item.
5. **Publish** (publisher role): validation runs again, then publishing continues in the background: the search index is built and embedded, and only then does the draft become the published version. You can leave the page; the progress shows when you come back. If it fails, the draft stays a draft and the embeddings computed so far are kept. Text that was embedded before is never embedded again.
6. **Discard** drops the draft.

**Four-eyes rule** (Settings page, off by default): whoever changed a draft cannot publish it; another publisher or admin must.

After a publish, agents' instructions are rebuilt and every tool reads the new version.

**Start from zero**: an admin can empty the ontology (Settings > Maintenance, type `EMPTY` to confirm). The current model is first saved as an Excel backup in `data/backups` (downloadable from the Excel card), then every version is deleted, and optionally the demo database too. Connectors, users, tokens and the activity log stay.

## The Excel template
Download it from **Changes & versions**: blank, or filled with any version (an export). Sheets:

| Sheet | One row per |
|---|---|
| Instructions | How to fill it in. |
| Model | A model property (name, version, standards, what each sensitivity class means...). |
| Entities | Entity. |
| Fields | Field (attribute or calculated field) of an entity. |
| Values | Allowed value of a field (value, label, note). |
| Filters | Named filter of an entity. |
| Checks | Data-quality check of an entity. |
| Relationships | Relationship (ids can be left empty: they are numbered automatically). |
| Examples | Query example (ids can be left empty). |

Conventions: required columns have a red header; every header has a help comment; closed lists are dropdowns; list cells separate items with ` | `; lifecycle states are one line per status column; parameters of examples are JSON. **Any column the template does not define is a custom property** (grey header in exports).

**Upload** (modeller role): the file (up to 20 MB) is first checked without creating anything. The report gives every problem with sheet, row and column, plus counts of what would be created. Only a file without errors becomes a draft; an existing draft is replaced only when you choose to. Export and re-import lose nothing.

An export by someone without the restricted entitlement leaves out the examples of restricted fields.

## Search
Hybrid search over the published model (or the draft, in the UI): every entity, field, calculated field, relationship, filter, check and query example is a **chunk** of text (custom properties and example questions included). Three rankings are merged with reciprocal rank fusion:
- **meaning**: vector similarity with the embedding model;
- **keywords**: PostgreSQL full-text search;
- **exact codes**: column names, abbreviations and record ids match exactly, shortest first.

Chunks are derived from the model and never edited; a chunk's text only changes when its item changes, so embeddings are reused across versions. For agents and Alfred, results follow the caller's entitlements: restricted fields and classes not granted are left out. Search in the UI covers the whole model's definitions, like the model pages. Searches by agents and Alfred that find nothing are kept as **questions nobody could answer** (Settings > Maintenance): a backlog of missing synonyms, terms or entities.

Without an embedding model (remote embeddings and no key), search is keyword and exact-code only.

## The demo database
A synthetic database generated from whatever model is published, to try everything before connecting real data. Admins build it on **Data → Database** (not while a connector is in use). It lives in the app's PostgreSQL as a separate database.

**Options**:
- **Scale**: `small`, `medium` or `large` (0.35×, 1×, 2.5× a base of 120 rows, times a factor per entity kind: reference tables are small, transactions and facts large; a code table gets one row per allowed code).
- **Seed**: the same seed and model give the same data.
- **Realistic values from the AI model**: one chat-model call per entity proposes fictional values for its free-text columns; only the model's names and descriptions are sent. Without a key, or unticked, values are guessed from types and column names.

**How the data is generated** (deterministic code, nothing about a particular model):
- Every entity is a table and every attribute a column (calculated fields are not columns).
- Keys are unique and follow the field's example or format (an example `AB12345` gives `AB` + 5 digits), else `<ENTITY>-000001`.
- Every foreign key points at a real row. Every parent gets a child before any gets two, then a long tail (a few busy parents, many quiet ones). Required (many-to-one) links are never empty; optional ones sometimes are.
- Chains agree: when a row reaches the same ancestor through two links (an order line's order and its subscription), it is the same row; a parent row is added when none fits.
- Self-links form shallow trees (up to 4 levels).
- Allowed values are used (the first ones more often); other values are guessed from type and column name (emails, phones, names, cities, amounts, rates, counts, dates in order: created before closed).
- Personal and restricted values are fictional.

The build runs in the background with progress, then validates the result; its findings appear in the Model check. Rebuilding replaces all data. When the published model changes, the page says the database was built from an older version.

The demo database has 8 read-only database roles, one per combination of the confidential, personal and restricted entitlements, each with column grants matching it and an 8-second statement timeout: a second line of defence under the query checks.

**Copy it to other databases** to try connectors on the same data (start `docker-compose.demo.yml` first), from the app container:
```sh
python -m studio.synth.export sqlite  /app/data/samples/demo.sqlite
python -m studio.synth.export duckdb  /app/data/samples/demo.duckdb
python -m studio.synth.export mariadb host=sampledb-maria port=3306 user=root password=... database_prefix=
python -m studio.synth.export clickhouse host=sampledb-ch port=8123 user=default password=...
```
SQLite has no schemas, so its tables are flat (the demo for "generate a model from this database"); the others keep the model's schemas.

## Connecting your data
A **connector** points the model at a database. Admins manage them on **Settings > Connectors**. One connector is active at a time; with none, the built-in demo database is used.

| Kind | Notes |
|---|---|
| PostgreSQL | Recommended. Structure, keys and small value sets read from the catalog and planner statistics, without scanning tables. SSL mode: `disable`, `prefer`, `require`, `verify-ca`, `verify-full`. |
| MySQL, MariaDB | Databases act as schemas; works with accounts that only have SELECT. |
| ClickHouse | No foreign keys: relationships are inferred from column names and marked unreliable. |
| SQLite, DuckDB | A file under `CONNECTOR_FILES_DIR` (default `/app/data`). DuckDB's access to other files is switched off. |

**Workflow** for each connector:
1. **Test**: database version, account, readable tables, and a warning if the account can write or administer.
2. **Read structure**: schemas (pick which), tables, columns, keys, row counts.
3. **Generate draft model** (pick the tables): entities from tables, fields from columns, relationships from foreign keys (or names), sensitivity guessed from column names on the safe side. Everything generated is marked *needs review*: it is a starting point to which modellers add meaning, filters, calculated fields and knowledge.
4. **Use for data**: makes it the active database. The app first reads the structure again and compares it with the published model; when tables of the model are missing, it shows the coverage and asks you to confirm.

**Options per connector**: statement timeout (default 15 s), row limit (default 500, also caps the console and agents), and whether agents may read rows at all (rows go to a language model; when off, `query_data` refuses and agents can still plan).

Passwords are stored encrypted and never shown again. A connector cannot point at the app's own database or at link-local or cloud-metadata addresses. Connections are read-only at the session level, with statement and idle timeouts. Activating, deactivating and every change go to the audit log.

## Checking the model against the data
The **Model check** (Data menu) runs the published model against the database in use and reports per group:
- **Structure**: every model column exists; every table has data; foreign keys as expected.
- **Relationships**: each join, how many rows link and how many are orphans.
- **Data-quality rules**: each entity check's violations.
- **Examples**: each worked example is run with sampled parameters until it returns rows.

Every result carries the SQL that produced it. Results show counts and statuses, never values. "Attention" items are usually questions for the modellers (the rule text and the physical schema do not fully line up) rather than data bugs.

On the demo database the result is stored with each build, the build's own findings are included, and a modeller can run it again; structure and joins always hold, while your data-quality rules may not (generated data does not know your business rules). On a connector the check runs live every time. On databases other than PostgreSQL, a statement the model's SQL cannot be translated into is reported as a warning.

## The knowledge library
The Library turns documents into a **topic wiki**: every document gets a verified **source wiki** (a standalone page written from it and checked against it), and approved source wikis are compiled into **topics**: pages by subject (a product, a policy, a process) that cite their sources, backed by atomic **claims** with exact quotes and linked to each other by typed **relationships**. Every layer is chunked, embedded and searchable by meaning and by words: the original text, every source wiki version, every topic version, the claims, the topics' identities and the relationships. It lives in **collections**, so what is known about HR, support playbooks or public product documentation never mixes. Nothing you add to the Library ever changes the ontology. Design notes: `docs/design.md`.

**The words**
| Term | Meaning |
|---|---|
| **Collection** | The unit of isolation. Documents, passages, topics, statements, links, vocabulary and jobs all belong to exactly one collection, and nothing is ever combined across collections. |
| **Source** | One document in a collection: an uploaded file, a captured web page or an interview. It has a status (waiting, working, ready, failed). |
| **Source wiki** (source page) | A standalone, factual page a model writes from the source, checked against it claim by claim. Every version is kept and indexed. |
| **Topic** | A page by subject, written from the full verified source wikis of one or many sources. Two documents about the same subject end up on one topic. Pages cite their sources as `[source: …]` chips. |
| **Claim** (statement) | One atomic fact of a topic, with its quote copied word for word from a source wiki, the source, the wiki version and the wiki passage it sits in. Claims are the provenance layer; they are superseded, never deleted. |
| **Relationship** | A typed link between two topics (*governs*, *depends on*, *part of*, *replaces*...) with why they relate. |
| **Vocabulary** | The topic types (with their meaning and page template) and relation types (with their inverse) a collection uses. Every collection starts from a rich starter ontology. |

### 1. Set up a collection
**Knowledge > Collections > New collection** (admins create; they may name the first manager).

- **Usage tag**: **Public** (may be shared with customers) or **Internal** (for the agent's own understanding: do not quote it or refer to it with customers). The tag is *guidance*, returned with every result, in each document tool's description and in the MCP instructions, so no agent depends on its prompt author remembering it. It is not an access rule.
- **Grants**: a user or an API token gets *read*, *write* or *manage* on the collection. *Read* searches and reads; *write* ingests, edits and approves; *manage* grants and revokes, changes settings, exports and deletes. Write and manage for users also need the **editor** role; tokens only ever read. A collection keeps at least one manager.
- **Switches** (collection settings): **Documents may be sent to the AI model** and **Agents may read this collection**. See [Feature switches](#switches-and-flags).
- **Hidden by default**: a collection you hold nothing on does not exist for you (the API answers 404). Admins see that collections exist so they can fix access, but not what is in them: an admin reads content only after granting themselves access, and that grant is logged.
- **Customer-facing agents** should get a token with grants on public collections only.

### 2. Add documents
**Knowledge > Sources**, in a collection you can write to (write access and the editor role).

- **Files**: drop or browse. PDF, Word (.docx), PowerPoint (.pptx), Excel (.xlsx), CSV/TSV, HTML, Markdown, text, JSON, XML and images, up to the upload limit. Legacy formats (.doc, .ppt, .xls, .rtf) are not accepted: save them in the new format. The same bytes cannot be added twice to a collection. A failed source can be retried.
- **Web pages**: **Add a web page** captures an address (kept as the page that was fetched, so the original can be downloaded). An address that is a file (PDF, Word...) is downloaded as that file. **Refresh** captures it again: unchanged means nothing is redone, changed means the new content is read again. The same content under two addresses is stored once. See [Fetching safely](#fetching-safely).
- **Interviews**: capture what a person knows by conversation. See [Interviews](#interviews).

Places are kept so passages can be cited: *Page 3*, *Slide 2*, *Sheet: Costs*. Headings become Markdown headings; web pages lose menus, footers and cookie banners; spreadsheets are read row by row with their column names. Scanned PDF pages and images are read by an **OCR model**: any vision-capable model on your AI endpoint (Settings > AI > *Model that reads images*, falling back to the assistant model; up to *Most scanned pages read per document*). Without one, such pages are skipped with a note, and a file with no readable text fails with the reason. Office files are checked for zip bombs before opening.

### 3. What happens to a document
1. **Read and split.** The text is cut into passages that fit the embedding model (about 75% of `EMBEDDING_MAX_TOKENS`), at paragraphs, with a little overlap, starting afresh at each page, slide or sheet. Each passage is embedded and indexed for keywords. This needs no AI endpoint, so even a collection that forbids AI is stored and searchable.
2. **Write the source page.** A *writer* model writes a standalone page that keeps every stated fact, number, condition and exception and never mentions the document itself. Long documents are written section by section.
3. **Check it.** A second *checker* model verifies the page claim by claim against the document, with the complete text when it fits and otherwise with tools (search the document, read a passage with its neighbours, list the outline). A FAIL that names no problem is asked for specifics and, if it still names none, the same version is checked again: the writer never amends blindly. A FAIL is also reviewed once by the checker itself, which drops entries it calls supported. If real problems remain, the writer corrects the page from the evidence and the checker looks again, up to *Correction rounds* (default 3).
4. **Approve or hold.** A page that passes is **approved**. One that still has objections goes to **needs review**: a person reads the objections next to the page and either approves it as it is or has it written again. Every version and every check is kept and can be opened.
5. **Build topics.** For an approved source, an *analyzer* model (with tools to search and read the collection's existing topics) decides which topics the source creates, updates or complements (at most 12). For each topic it extracts atomic claims, each with a quote that must appear word for word in the approved source wiki (a claim whose quote is not there is dropped). The *topic writer* then writes or amends each page from the **full verified source wiki**, the candidate and its claims, following the topic type's template; it can search and read related topics and **other approved source wikis** of the collection, keeps every cited fact the new evidence does not supersede, and cites sources. Each claim records the wiki version and passage its quote comes from. Every page version is chunked and embedded, the topic's identity vector is rebuilt from its claims, and links are embedded with their context.
6. **Link to the model.** In the same pass the analyzer says which model entities each topic is about (see [The bridge](#the-bridge-to-the-data-model)).
7. **The review gate.** Most updates publish at once; contradictions and probable duplicates wait (see [Review](#4-review)).

The wiki models see only the one document of the job; the topic models see the collection's topics and approved source wikis, never another collection's. Which model does which job is set per job (Settings > AI); a job without its own model uses the assistant model. If the topic writer fails, a plain page is built from the candidate so nothing is lost. The same claim found again in another source is attached to the existing one (numbers stay stable). Reading a source again supersedes the claims only it supported and states them afresh; a topic it no longer mentions gets a **surgical rewrite** that checks each withdrawn claim against the remaining sources' wikis and removes only what nothing else supports.

**Editing and downloads.** With write access and the editor role, **Edit the page** (or *Write the page by hand* where AI is off) saves a new version marked *edited*, with your name. With AI available it is checked once against the document, not rewritten: a pass approves it, otherwise it waits for review. Without AI, saving it is the approval. Anyone who can read the collection can download the original file and the page (Markdown, any version).

### 4. Review
**Knowledge > Review** is the collection's queue: everything that waits for a person. The sidebar badge counts what waits for you across your collections.

| Item | What you do |
|---|---|
| **Contradiction**: new evidence directly contradicts a statement on the existing page | The review screen shows the published page beside the page as it would become, the new statements with their quotes and the statement being contradicted. **Approve** (the old statement is superseded), **reject** (the change is discarded), or leave it. |
| **Probable duplicate**: a new topic very similar to an existing one (*Similarity at which a new topic is held*, default 0.92) | **Approve**, **reject** or **merge** it into the existing topic. |
| **Source page with objections** | Approve as it is, or have it written again. |
| **Suggested link to the data model** | Accept or reject in place. |
| **Document that could not be read** | Retry or remove. |
| **Vocabulary proposal** | The collection's manager accepts or rejects (an editor sees it as waiting for the manager). |

A held change is written but not published: its statements stay *pending*, readers see the old page and agents see nothing of it. Two held changes on one topic never build on each other's unapproved work. Everyone who can write in the collection sees the same queue, so whoever is free picks it up; **readers do not see it** because held changes are unpublished. Interviews are different: the conversation belongs to the person who held it.

### 5. Read and search
- **Topics** (Knowledge > Topics): the article with source chips and links to related topics, and beside it every claim with its quote, source, source wiki version and the wiki passage it sits in. Aliases, related topics (with their relation), the data-model links and the version history are on the page.
- **Search** (Knowledge > Sources): a search box for the selected collection combines meaning and words; each result shows its source, place, section and similarity and carries the collection's usage note. *Smallest similarity* (Settings > Library) can hide weak matches; it is off by default because models differ. Search is always inside one collection: the database itself refuses a passage that names a source of another collection.
- **The search box in the top bar** finds topics, documents and collections across everything you may read, each result showing its collection and usage tag.
- **Graph** (Knowledge > Graph): the topics of one collection and their links (colour = type); tick *Also show the data-model entities these topics are about* to add the entities. The model's own graph can overlay the topics of the collections you can read.

### 6. Keep it healthy
- **Merging**: any topic can be merged into another (topic page > *Merge into another topic*). Its statements are renumbered into the target (a statement the target already makes keeps one copy with both pieces of evidence), its title and aliases become aliases, its links and sources move, the target's page is written again and the old address keeps working as a redirect.
- **Revising (the lint pass)**: when a source is deleted, the claims only it supported are superseded (their quotes are kept) and the topics that relied on it are marked *needs revision*. A background job runs the surgical rewrite: the writer checks each withdrawn claim against the remaining sources' wikis and removes or softens only what nothing else supports. A topic with no source left is archived (not listed, not found; its history stays).
- **Vocabulary** (Knowledge > Vocabulary): the collection's manager adds kinds, edits their meaning and (for topic types) the page template the writer follows, accepts or rejects what the analyzer proposed, deletes a kind nothing uses, or **merges** one kind into another (every topic or link of the first becomes the second). `Topic` and `related_to` are the fallbacks and cannot be removed. When nothing fits, the analyzer proposes a new kind and the closest existing kind is used meanwhile. Nothing one collection adds reaches another, and the model's ontology is never touched.
- **Health** (Knowledge > Health): a check without any AI model: held changes, topics needing revision, topics with no source evidence or no published page, active claims citing no source, source pages with objections, failed sources, approved sources not yet turned into topics, isolated topics, links to archived topics, statements whose quote is no longer on their source's page, and pairs of very similar topics that may be one subject. The last 20 runs are kept.
- **Export** (Collections > *Export*, managers; audited and rate limited): the whole collection as one zip. It holds the **original files** as uploaded, what is known about each source, every source page with all versions and checks, every topic as Markdown with front matter plus a JSON with every claim, its exact quote, the source and the source wiki version and passage it comes from, the links, the vocabulary, the collection's settings and a manifest that lists anything that could not be included. Held changes and interviews are left out. It is an archive and handover format; there is no import yet.

### The bridge to the data model
A topic can point at model entities (and fields): *the Pro plan* is a kind of *Product*; a refund policy governs *Invoice*. The pointer is stored **inside the topic's collection**, never in the model. The model gets no new entity and no new relationship, and only people and tokens that can read the collection ever see the link: if you cannot read it, there is nothing to see and nothing broken.

- **Made while reading.** The analyzer is shown the model's entities (name, domain, a short definition: structure only, never data or field details; a model of more than 200 entities is first narrowed to the ones that look like the source) and says, for each topic, which entities it is about, with a relation (*is a kind of / is about / governs / concerns*) and a confidence. A document about one product becomes a topic already linked to the product entity, with no extra model call. It chooses every entity the topic is genuinely about (usually one or two, a safety limit of 12). At least 70% counts at once; less waits as *suggested* until a person accepts it. A rejected link is not proposed again.
- **By hand.** Topic page > *In the data model* > *Link to an entity*; fields can be linked too, but never personal, confidential or restricted ones. **Suggest with AI** runs the analysis on an older topic.
- **Where it shows.** The topic page; the **entity page** (*Related documents*, grouped by collection with its usage tag); the model graph (tick *Show the topics of my document collections*); and for agents, `library_search` and `library_topic` list the entities a topic is about, `describe` of an entity adds `related_documents`, and `plan_query` lists the relevant topics.
- Links are resolved against the published model when read, with the reader's own entitlements; one whose entity has left the model is flagged. What the model says a topic *is* (its *is a kind of* links) is shown as the topic's kind next to the collection's own topic type.

### Interviews
**Knowledge > Interview** captures what a person knows by asking them questions, one at a time, the way a curious colleague would. You say what you want to document in your own words; an interviewer model (Settings > AI > *Model that conducts interviews*) asks, reflects back what it heard, follows threads, asks for specifics and checks stored facts against what you say next. After every exchange a second step extracts the **facts** established (one per sentence, with a category such as trigger, step, decision, exception); they are the interview's long-term memory and you can correct or remove any of them. **Write the draft** produces a document from the facts (every fact must appear, nothing else is added); you can edit it and save new versions. **Add to the collection** turns it into a source (kind *interview*) that is read, indexed and turned into topics like any document. Because a person reviewed the draft, it is its own source page: the writer and checker are not run on it. The conversation belongs to whoever held it (a collection's manager can see them all). It needs write access, the editor role and a collection that allows AI, and counts against the heavy-action rate limit.

### Fetching safely
Only public `http://` and `https://` addresses. The name is resolved once and every address it has must be public (private, loopback, link-local, metadata, reserved, carrier-grade and unique-local ranges are refused, including odd spellings such as `http://2130706433/` or IPv4-mapped IPv6). The connection is made to exactly that IP, with the original name for the Host header and the TLS certificate, so a name that answers differently a second time cannot redirect the request (DNS rebinding). Every redirect is checked again (at most 5), the size is capped (*Largest web page or file captured*), and addresses with a user name or password are refused. To read an intranet, list its hosts or networks in Settings > Web capture > *Internal hosts web capture may reach*. For pages built by JavaScript, add the [browser service](#optional-services); its browser never touches the network itself, because every request it makes is answered by the same safe fetch.

### Where files live and background work
- **Storage** (Settings > Storage): `local` (default) keeps files in `data/library` inside the mounted data folder; `s3` uses any S3-compatible bucket (see [Optional services](#optional-services)). Files are keyed `<collection>/<source>/<name>`: deleting a source deletes its file, deleting a collection deletes everything under its prefix, and on a versioned bucket every old version and delete marker goes too. **Test the storage** checks the saved values. Connectors cannot open files under `data/library`.
- **Background work**: processing runs as jobs in a queue in the database (claimed so that no two workers take the same job; a job whose worker dies is picked up again; failures retry with a growing pause, then show their error). By default the web app runs the workers itself (*Background workers in this process*); for heavy use add the [worker container](#optional-services). Job payloads hold ids only, never document text.
- **Activity**: each document's processing is a session with source *Library* on the Activity page: its steps, how long each took, attempts, errors and the model calls with their cost. Sessions show an id, never the file name. **Admin > Activity > Audit trail** lists who created, changed or deleted collections, who granted or removed access, self-grants, uploads and removals of sources (by id), and role and setting changes.


## Agents: the MCP server
The MCP endpoint is `/mcp` (streamable HTTP). Agents authenticate with a bearer token created on **Users & access > Tokens & MCP**; each token carries its own entitlements for the model and data (see [Access control](#access-control-and-data-protection)) and, separately, its **collection grants** for documents (Knowledge > Collections). A new token is shown once; tokens are stored hashed, and the page shows when each was last used. Revoking deletes it.

**Connect**, for example with Claude Code:
```sh
claude mcp add --transport http seamory https://<your-host>/mcp --header "Authorization: Bearer <token>"
```
Any MCP client that supports streamable HTTP and a bearer header works the same way.

**Sixteen tools, all read-only and idempotent.** The first six cover the ontology and the data and always appear. The ten `library_` tools (agent retrieval over the knowledge base, collection by collection) appear only to a token that holds a grant on at least one collection (a token without grants sees six tools and no trace of the Library). All read the published version only.

| Tool | Parameters | What it returns |
|---|---|---|
| `get_access` | none | Who the caller is, which sensitivity classes it may see, whether it may query data, and the SQL dialect of the connected database. |
| `search` | `query`, optional `types`, `limit` (1-25, default 10) | The best matching entities, fields, calculated fields, filters, relationships and query examples for words or exact codes. |
| `describe` | `name`, optional `kind`, `include_attributes` | Anything by name: an entity with its fields, allowed values, filters, checks, calculated fields, links, custom properties, the query examples that apply and `related_documents`; a field (`entity.column`); a calculated field, filter or query example; or a word the model does not know but the documents you may read do (with the entities the topic is about). With no name, a list (the overview of the whole model for `kind='entity'`). |
| `join_path` | `from_entity`, optional `to_entity` | With a target: the best path with exact join conditions, how far each link can be trusted, an all-confirmed alternative, fan-out warnings and related documents about either end. Without: every entity directly linked, with join, cardinality and reliability. |
| `plan_query` | `question`, `entities`, optional `grain` | For a data question: per entity the table, the most useful columns with meaning and allowed values, how it joins to the grain, the named filters the question's words match (pre-filled), related calculated fields and examples, entities the question names but were not listed, a runnable one-statement SQL skeleton in the database's dialect with one pre-aggregated subquery per one-to-many branch (no fan-out), and `documents`: every topic from the collections you may read that is about these entities or matches the question, as brief entries (title, a line of summary, why it matched, where to open it, the collection's usage tag), with a high cap that reports how many more matched. Deterministic, no language model involved. |
| `query_data` | `sql` | Runs one read-only statement under the caller's entitlements. See below. |
| `library_context` | `query`, optional `collection`, `top_n` (1-10, default 5), `depth` (1-3) | **Start here.** Searches the topic pages by meaning and words, reranks, and returns per collection the **full Markdown** of each matching topic with the matched passage and its neighbours (`context_chunks`), the graph of related topics, the typed relationships that match the question, and `answerable` (false: the documents do not cover it, do not answer from your own knowledge), with the size in tokens. |
| `library_search` | `query`, optional `collection`, `limit` (1-20) | Lightweight discovery: slugs, titles, summaries, scores and the best matching chunk, no page body. |
| `library_topic` | `collection`, `topic` | One full topic page: Markdown (sources cited as `[source: <id>]`), aliases, typed links both ways, its sources with the wiki version used, the model entities it is about. |
| `library_topic_claims` | `collection`, `topic` | Every active claim of the topic with its exact quote, the source, the wiki version and the wiki passage it comes from. |
| `library_topic_sources` | `collection`, `topic` | The documents behind the topic, the wiki version each was used at and whether it is still the approved one. |
| `library_traverse` | `collection`, `topic` (one or a list), optional `depth` (1-3), `relations`, `max_topics` | BFS over typed links in both directions: topics with summaries and edges with why they relate. Links never leave the collection. |
| `library_source_search` | `query`, optional `collection`, `limit` | Searches the approved source wikis by meaning and words: documents with the passage that matched. |
| `library_source` | `collection`, `source`, optional `query`, `around`, `window` (0-3), `original` | Reads a source's approved wiki: the whole page, a search inside it (best passages with their neighbours), or a passage and its neighbours; `original=true` reads the original extracted text instead. |
| `library_vocabulary` | optional `collection` | The topic types and relation types (with inverses) of a collection. |
| `library_collections` | none | The collections you may read, each with its usage tag, what it holds and its number of topics. |

**How retrieval works.** Chunks are how the system *finds*, topic pages are what it *returns*: `library_context` matches the question against chunks of the published topic pages (and the topics' identities), keeps the best chunk per topic, reranks, and hands back the whole page with the matched passage, so an agent gets the full argument rather than a fragment. From there, `library_topic_claims` and `library_topic_sources` give provenance, `library_traverse` the graph, and `library_source_search` / `library_source` the verified source wikis for a detail a topic does not carry. Alfred shrinks an over-long result in layers (passage windows first, the pages are kept) and says what it left out.

**Which tool for which question.** A question about data goes through `plan_query` and `query_data`. A question about what the business says (a rule, a definition, a procedure) goes to `library_search`. The server instructions say most questions need only one of the two; when a data question also touches company rules, `plan_query` hands the agent the relevant topics so it knows what to open.

**Usage tags travel with the content.** Every collection block in a result carries its `usage` (`public` or `internal`) and a one-line note, in the data itself, in each document tool's description and in the server instructions: *public* may be shared with customers; *internal* is for the agent's own understanding and must not be quoted or referred to in customer text. The tag is guidance; the grants decide what a token can read.

**What `query_data` returns**:
- `columns`, `row_count` and `rows`: at most `AGENT_MAX_ROWS` rows (default 100), or the connector's lower row limit.
- When the result has more rows: `truncated`, `total_rows` (the full count, from a count of the same query under the caller's rules; left out if it cannot be counted) and `truncated_note`. The note says how many rows there are, that only the first ones are returned, whether they are in no particular order (no `ORDER BY`), and that rows are not paged: to cover all of them, count or aggregate in SQL or narrow the query.
- `filter_matches`: when a filter on a descriptive text column selects records whose key the query then uses (a lookup by name, for example) and it matched more than one distinct record: the entity, column, filter, how many records, up to 5 examples and a note. Information only; a column the caller cannot read in clear is never probed.
- `masked_columns` and a note when some columns came back as hashes; `note` when the data is the synthetic demo database.
- On an error: the database's message and, for an unknown column or table, where that name does exist.

The notes come before the rows, so a model that cuts a long result by size still reads them.

**What a client gets without anything in its own prompt**:
- **Instructions** sent at connection, written as how to read a question: (1) understand the words through the ontology (`describe` gives what "active", a plan name or an abbreviation means here; never assume), (2) route it: what the organisation *says* goes to the documents (`library_context`), what the data *shows* goes to `plan_query` then one `query_data` statement, what something *means* goes to `describe` / `search` / `join_path`, and a rule applied to the data uses both (the rule from `library_context`, the entities from the topic's `model` list, then one planned statement), (3) answer with evidence. Plus the data workflow, the document workflow (`answerable`, usage tags) and how to answer. Rebuilt on every publish.
- **Self-sufficient tool descriptions**: each says when to use it, what comes before and after, and the rules that matter, because many clients never show the instructions to their model.
- **`next` hints in results**: what to call next (for example `plan_query` after `describe`, `library_topic_claims` after `library_context`, other words to try when the documents do not answer). The document tools are only suggested to a credential that can read a collection.
- **Resources**: `ontology://guide` (the same guide).
- **Prompts**: *Explore a topic of the model*, *Answer a data question with one query*, *Answer from the company documents* and *Answer a question that needs the documents and the data*.
- **Tool annotations** mark every tool read-only, so clients can approve them automatically.

**What an agent can reach in the Library**: only collections its token holds a grant on, and only if the collection allows agents to read it. A collection that does not exist and one the agent may not read give the same answer. Calls are logged on the Activity page like the model tools.

Every call is recorded in the Activity log (one session per token, a new one after 30 minutes without calls) and counts against the per-client rate limit.

## Alfred, the built-in assistant
Alfred answers questions about the model, the data and the documents inside the app, using the same tools as external agents (the six model tools except `get_access`, which it does not need, plus the ten `library_` tools, starting from `library_context`) and the same guide, under the signed-in user's entitlements and collection grants. Open it from any page with **✦ Ask Alfred** (a small window; icons for a new chat, full screen and close), or full screen for a chat-first experience.

- It knows which page you are on ("what does this field mean?" on a field page).
- It decides when it needs the documents: a data question uses the model and the data only, a policy question searches the Library, and a general question is answered without any tool.
- It streams its answer, shows each tool it calls, and lists the entities, fields and topics it used as links.
- Alfred reads a collection only if it allows agents to read it **and** *Documents may be sent to the AI model* is on, because Alfred sends what it reads to its model.
- It has a set number of lookups per question (`ALFRED_MAX_LOOKUPS`, default 10); when they run out, it answers from what it found. Tool results longer than `ALFRED_TOOL_RESULT_CHARS` are cut before the model reads them.
- The conversation is kept in the browser tab (the last 40 messages) until you clear it or close the tab. Each question sends the conversation's text, not earlier tool results.
- Answers never load images or outside content by themselves, and instructions found inside data, model text or documents are treated as content, not commands. Internal collections are never quoted in customer-facing text.
- Limited to `ALFRED_PER_MIN` questions per user per minute.

Alfred uses a chat model through the **AI endpoint** (Settings > AI): any OpenAI-compatible server (OpenRouter, a LiteLLM gateway, Ollama, vLLM...). Set the address, the key (optional for servers that need none), the model, reasoning and temperature there, and press **Test the endpoint**. OpenRouter-only request fields are sent only when the address is OpenRouter's. Without an endpoint, Alfred is off and everything else works.


## Access control and data protection
**Sensitivity classes.** Every field is one of:

| Class | Typically | Who sees it |
|---|---|---|
| internal | the structure and meaning of the model | everyone |
| confidential | financial and other business-sensitive values | holders of the class |
| personal | names, emails, phone numbers of persons | holders of the class; others get hashes where the field is masked |
| restricted | identity numbers, credentials and similar | only holders of the restricted entitlement, granted explicitly |

**Entitlements** are four switches per credential: `confidential`, `personal`, `restricted`, and `data` (may query data at all).
- New **tokens** start with confidential and personal, without restricted and **without data** (structure only until an admin grants it).
- New **users** start the same but with data.
- Restricted is always off unless granted explicitly; the page asks for confirmation, and **every grant is logged**.
- A user's entitlements govern what Alfred, the data browser and the SQL console show them. The model pages show every signed-in user the model's definitions (not data).

**Agent access per field**:
- `full`: readable by whoever holds its class.
- `masked`: readable as a **stable hash** (`h_` + 12 hex characters) by callers without the class. Equal values give equal hashes, so masked columns can still be selected, counted, grouped, ordered and joined, but never filtered, compared or transformed (which would let a caller recover the value).
- `none`: never readable without the restricted entitlement. These fields are also left out of search and of everything agents receive.

**Enforcement.** Every query from an agent, from Alfred or from the SQL console is parsed, every column traced to its real table through aliases, CTEs and subqueries, and checked against the caller's rights. The statement that runs is regenerated from the checked tree, never the text that was sent. Rules: one read-only statement; only tables the model describes; no system, file, network or administration functions (each database driver lists its own); restricted columns never without the entitlement; a hashed column only in the outermost select or in counts, groups, orders and joins. SQL written in the model itself goes through the same checks.

The database account is the second line of defence: always read-only, with timeouts; on the demo database also column grants per entitlement combination.

## Administration
Users and roles, the switches, observability, every setting, the proxy and day-to-day operations.


### Users, roles and sign-in
| Role | Can |
|---|---|
| viewer | Explore, search, browse data, use Alfred and the SQL console under their entitlements. |
| modeller | Also edit drafts, upload Excel, run the Model check again. |
| publisher | Also publish and discard drafts. |
| admin | Also manage users, tokens, connectors, settings, the demo database, the activity log, and empty the ontology. |

Each of these roles can do everything the ones before it can. Separately, an **editor** role can be ticked on any user (an admin always has it): it is for the Library, not the model: together with a *write* or *manage* grant on a collection it allows ingesting, editing and approving documents. A user's roles are the rung they hold plus the extras, shown by `/api/me` as `roles`. Admins add users (or update an existing one) on **Users & access > Users** with a role, entitlements and a password, and can change a user's roles later without touching the password. The last admin cannot be demoted.

- Passwords need at least 12 characters and cannot be the username; they are stored with scrypt.
- A password set by an admin is temporary: the user picks their own at the next sign-in, and their other sessions end.
- Changing your password ends your other sessions.
- Sign-in is locked for 15 minutes after 5 failures for a username or 30 from one client address; while locked, even the right password is refused. An unknown username takes as long as a wrong password.
- Sessions are cookies that last `SESSION_HOURS` (default 12).
- The first admin comes from `INITIAL_ADMIN_USERNAME` / `INITIAL_ADMIN_PASSWORD`, created only when there are no users.

### Switches and flags
Every on/off or mode choice in one place. Settings marked *page* are on the Settings page (and can also come from `.env`); the rest live where the object is.

| Switch | Where | Default | Effect |
|---|---|---|---|
| **Library on or off** (`LIBRARY_ENABLED`) | Settings > Library | on | Off hides the Knowledge menu, the document tools and the API; nothing is deleted. |
| **Documents may be sent to the AI model** | Collection settings | on | Off: the collection is stored, split and searchable by words and meaning (local embedding), but no model ever sees it: no source page, no OCR, no topics, no link suggestions, and Alfred cannot read it. |
| **Agents may read this collection** | Collection settings | on | Off keeps Alfred and MCP clients away from the collection entirely, whatever the grants. |
| **Usage tag** (public or internal) | Collection settings | internal | Guidance returned with every result so agents know whether they may quote the content to customers. Not an access rule. |
| **Collection grants** (read, write, manage) | Collection access table | none | The only thing that decides who and what can read, write or manage a collection. Tokens are read only. |
| **Editor role** | Users | off | Needed (with a write or manage grant) to ingest, edit and approve documents. Admins always have it. |
| **Entitlements** (confidential, personal, restricted, data) | Per user and per token | see [Access control](#access-control-and-data-protection) | What each credential may see of the model's data. Restricted is never on by default and every grant is logged. |
| **Agents may read rows** | Per connector | on | Off: `query_data` refuses, and agents can still plan queries. |
| **Four-eyes rule** | Settings > General (page only) | off | Whoever changed a draft cannot publish it. |
| **Settings locked** (`SETTINGS_LOCKED`) | `.env` | off | The Settings page becomes read-only; only `.env` counts. |
| **Reasoning** (`CHAT_REASONING`) | Settings > AI | off | Let reasoning models think first: better on hard questions, slower and dearer. |
| **Storage driver** (`STORAGE_DRIVER`) | Settings > Storage | local | `local` or `s3`. Choose before the first upload. |
| **Render pages with the browser service** (`FETCH_RENDER`) | Settings > Web capture | auto | `never`; `auto` (only pages with almost no text without JavaScript); `always`. Needs the browser service address. |
| **Background workers in this process** (`LIBRARY_WORKERS`) | Settings > Library | 2 | `0` when a separate worker container does the work. |
| **Service role** (`SERVICE_ROLE`) | `.env`, set by `docker-compose.worker.yml` | all | `all`: web and workers in one process; `api`: web only. |
| **Embedding provider** (`EMBEDDING_PROVIDER`) | `.env` | local | `local` runs in the container; a remote model needs `EMBEDDING_MODEL` and `EMBEDDING_DIM`. |
| **Reranker** (`RERANK_URL`) | Settings > AI | off | Sharpens Library search when set; if it fails the order simply stays. |
| **Keep tool responses in the log** (`ACTIVITY_RESULT_CHARS`) | Settings > General | 8000 | `0` keeps only the summary. |
| **Realistic values from the AI model** | Demo database build | off | One model call per entity proposes fictional values; only names and descriptions are sent. |


### Observability
**Activity** (admins, Admin → Activity):
- **Sessions**: every Alfred conversation and every agent session (an MCP token's calls with no gap over 30 minutes), newest first, with source, who, number of questions, the latest question and when it was last active.
- **Steps** of a session: the question, each model call (time, first token, tokens in and out, cost, tools requested), each tool call (arguments, time, outcome, answer size, rows, and the response itself, cut to `ACTIVITY_RESULT_CHARS`), the answer, and whether Alfred ran out of lookups. A long-running chat shows a break wherever it paused for more than 30 minutes, with times restarting after it. Responses load when opened and can be copied.
- **Summaries** per tool and per model: calls, errors, time, tokens and cost over the last days.
- A response is exactly what that caller received, so it holds data only as their entitlements allowed it (masked columns stay hashed); set `ACTIVITY_RESULT_CHARS=0` to keep only the summary.
- Steps older than `ACTIVITY_DAYS` are purged every hour, and sessions with nothing left go with them. Writing to the log never slows or breaks a request.

**Audit log** (Changes & versions): every model edit, import, publish, discard, grant, connector change and activation, setting change and reset, with who and when.

**Questions nobody could answer** (Settings > Maintenance): agent and Alfred searches that found nothing, newest first.

**Search index** (Settings > Maintenance): embedding model and Alfred status, chunks without an embedding, and **Rebuild search index** (fills gaps, for example after changing the embedding model; also done automatically at startup).

**Health**: `GET /api/health` returns `{"ok": true}` without signing in, for load balancers and monitoring.

### Settings and configuration reference
All settings are environment variables, normally in `.env`. The ones marked ⚙ can also be changed by an admin on the **Settings** page (Admin → Settings):
- The change applies on the next request, with no restart. Rate limits switch over at once; the sign-in duration applies to new sign-ins. Other app processes pick a change up within 5 seconds.
- Each value comes from, in order: the page, `.env`, the built-in default. Each row shows which; **Reset** removes the page value and shows the `.env` value it goes back to.
- Values are checked (type and range); a bad value is refused with a message.
- Every change goes to the audit log with the old and new value.
- The OpenRouter key set on the page is stored encrypted and never sent back: the page only says whether one is set.
- `SETTINGS_LOCKED=true` makes the page read-only and ignores values stored there, for teams that keep configuration in files. The four-eyes rule has no variable and stays editable on the page.

Settings the app needs before it can reach its database, the ones that describe the deployment (proxies, ports) and the embedding model (changing it means re-embedding) are environment only.

| Setting | Default | Range | Meaning |
|---|---|---|---|
| `DB_PASSWORD` | required | | PostgreSQL superuser, used only to create the app's account on first start. |
| `APP_DB_PASSWORD` | required | | The app's own database account (`studio_app`). |
| `SECRET_KEY` | required | | Signs sessions and derives the keys that encrypt connector passwords and the OpenRouter key and salt masked values. Changing it later means re-entering connector passwords and the key. |
| `INITIAL_ADMIN_USERNAME`, `INITIAL_ADMIN_PASSWORD` | `admin`, required | | The first administrator, created on first start. |
| `HTTP_BIND`, `HTTP_PORT` | `127.0.0.1`, `8080` | | Where the app listens (docker-compose). |
| `CLIENT_IP_HEADER`, `TRUSTED_PROXIES` | `X-Forwarded-For`, private networks | | How to find the real client behind a proxy (see below). |
| `RATE_PER_S`, `RATE_BURST` ⚙ | `20`, `60` | 1-1000, 1-10000 | API and MCP requests per client address: steady rate and burst. |
| `LOGIN_PER_MIN` ⚙ | `10` | 1-100 | Sign-in attempts per client address per minute. |
| `ALFRED_PER_MIN` ⚙ | `20` | 1-600 | Alfred questions per user per minute. |
| `INGEST_PER_MIN` ⚙ | `20` | 1-600 | Heavy actions one user may start per minute: uploads, web addresses, refreshes, page regeneration and edits, topic revisions, link suggestions, interview drafts, exports. |
| `SESSION_HOURS` ⚙ | `12` | 1-720 | Browser session lifetime. |
| `ACTIVITY_DAYS` ⚙ | `30` | 1-3650 | How long the activity log is kept. |
| `ACTIVITY_RESULT_CHARS` ⚙ | `8000` | 0-100000 | How much of each tool response the activity log keeps (`0`: none, only its size). |
| `LLM_BASE_URL` ⚙ | OpenRouter's address | | The AI endpoint: any OpenAI-compatible server. Normally set on the Settings page. |
| `OPENROUTER_API_KEY` ⚙ | empty | | Key for the AI endpoint (the name is historical). Alfred, the AI step of the demo database, and remote embeddings. Optional for a server that needs none. |
| `CHAT_MODEL` ⚙ | `openai/gpt-6-luna` | | Assistant model: Alfred's, and the default of other AI features (any model with tool calling). |
| `CHAT_REASONING` ⚙ | `false` | | Let reasoning models think first (better on hard questions, slower and dearer). |
| `ALFRED_TEMPERATURE` ⚙ | `0.2` | 0-1 | Alfred's temperature. |
| `ALFRED_MAX_LOOKUPS` ⚙ | `10` | 3-20 | Tool rounds per question before Alfred answers with what it found. |
| `ALFRED_TOOL_RESULT_CHARS` ⚙ | `16000` | 2000-64000 | Tool results longer than this are cut before the model reads them. |
| `AGENT_MAX_ROWS` ⚙ | `100` | 10-1000 | Rows `query_data` returns to Alfred and MCP clients. A longer result gives the total (`total_rows`) and a note suggesting counts, groups or filters; rows are not paged. |
| `CONSOLE_MAX_ROWS` ⚙ | `500` | 10-5000 | Rows in the SQL console (a connector can set a lower limit). |
| `LIBRARY_ENABLED` ⚙ | `true` | | Turns the Library on or off (nothing is deleted). |
| `LIBRARY_WORKERS` ⚙ | `2` | 0-8 | Documents processed at once by the web process; `0` when a separate worker container does the work. |
| `LIBRARY_MAX_UPLOAD_MB` ⚙ | `100` | 1-2000 | Largest upload. |
| `STORAGE_DRIVER` ⚙ | `local` | `local`, `s3` | Where original files are kept. |
| `S3_ENDPOINT`, `S3_BUCKET`, `S3_REGION`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` ⚙, `S3_PATH_STYLE`, `S3_PREFIX` | empty | | Any S3-compatible bucket; the secret key is stored encrypted. |
| `MODEL_WRITER`, `MODEL_VERIFIER`, `MODEL_ANALYZER`, `MODEL_TOPIC_WRITER`, `MODEL_INTERVIEW`, `MODEL_OCR` ⚙ | empty | | A model per Library role; empty uses the assistant model. |
| `MAX_AMENDMENT_ROUNDS` ⚙ | `3` | 0-8 | Correction rounds for a source page before it goes to a person. |
| `WRITER_SECTION_TOKENS` ⚙ | `8000` | 1000-100000 | Longest text written in one go; longer documents are written in sections. |
| `OCR_MAX_PAGES` ⚙ | `100` | 1-1000 | Most scanned pages read per document. |
| `DUPLICATE_SIMILARITY` ⚙ | `0.92` | 0.5-1 | Similarity at which a new topic is held as a probable duplicate. |
| `LIBRARY_MIN_SIMILARITY` ⚙ | `0` | 0-1 | Smallest similarity for a passage to count as a match. |
| `RERANK_URL`, `RERANK_MODEL`, `RERANK_API_KEY` ⚙ | empty | | Optional Cohere-style reranker for Library search. |
| `FETCH_ALLOWED_HOSTS` ⚙ | empty | | Internal hosts or networks web capture may reach anyway. |
| `FETCH_MAX_MB` ⚙ | `25` | 1-200 | Largest web page or file captured. |
| `FETCH_RENDER` ⚙ | `auto` | `never`, `auto`, `always` | Use the browser service for pages built by JavaScript. |
| `FETCHER_URL`, `FETCHER_TOKEN` ⚙ | empty | | Address and shared secret of the optional browser container. |
| Four-eyes rule ⚙ | off | | Page only: whoever changed a draft cannot publish it. |
| `SETTINGS_LOCKED` | `false` | | `true`: the Settings page is read-only and only `.env` counts. |
| `EMBEDDING_MAX_TOKENS` | `512` | | How much of a passage the embedding model reads. The Library cuts documents to about 75% of it. |
| `EMBEDDING_PROVIDER` | `local` | | `local` runs the model in the container (about 30 ms per query, no key). `openrouter` uses a remote model through OpenRouter. |
| `EMBEDDING_MODEL`, `EMBEDDING_DIM` | `BAAI/bge-base-en-v1.5`, `768` | | The embedding model and its vector size, any size pgvector stores (1 to 16000). The database tables are created for that size. Starting with a different size converts them, empties the vectors and recomputes them in the background. |
| `CONNECTOR_FILES_DIR` | `/app/data` | | Where SQLite and DuckDB files may live. |
| `SAMPLEDB_PASSWORD`, `SAMPLE_MARIA_PASSWORD`, `SAMPLE_CH_PASSWORD` | empty | | Only for the demo databases of `docker-compose.demo.yml`. |

**Advanced** (rarely changed; not in `.env.example`):

| Setting | Default | Meaning |
|---|---|---|
| `DATABASE_URL` | set by docker-compose | The app's connection to its PostgreSQL, as the least-privileged `studio_app` account. |
| `DATA_DB_NAME` | `semantic_data` | Name of the demo database inside the app's PostgreSQL. |
| `DATA_DIR` | `/app/data` | The mounted data folder. The Library keeps its files in `library` below it. |
| `SERVICE_ROLE` | `all` | `all`: web and job workers in one process; `api`: web only; the worker container runs `python -m studio.worker`. |
| `OPENROUTER_BASE` | `https://openrouter.ai/api/v1` | Default of `LLM_BASE_URL`. |
| `EMBEDDING_QUERY_PREFIX` | the prefix `bge` models expect | Text put before a search query when embedding it; set it for a model that expects another. |
| `MODEL_CACHE` | `/app/models` | Where the local embedding model is kept. |
| `FRONTEND_DIR` | the built UI in the image | Where the UI files are served from. |

### Running behind a proxy
The app sets its own security headers and rate limits, so it is safe with no proxy in front, and a proxy only adds a layer. To expose it, terminate TLS in your proxy (nginx, Traefik, HAProxy, a cloud load balancer) and point it at the app. The app needs the real client address for rate limits and sign-in protection; it reads a forwarded header **only when the request comes from a trusted proxy**:

| In front of the app | `CLIENT_IP_HEADER` | `TRUSTED_PROXIES` |
|---|---|---|
| nothing (app exposed directly, `HTTP_BIND=0.0.0.0`) | *(empty)* | *(unused)* |
| one proxy | `X-Forwarded-For` | the proxy's address or network |
| nginx with `proxy_set_header X-Real-IP $remote_addr` | `X-Real-IP` | the proxy's address |
| Cloudflare, then a proxy | `CF-Connecting-IP` | the proxy's network |

With `X-Forwarded-For`, the client is the rightmost address that is not a trusted proxy, so a client cannot pose as someone else by writing the header. The default trusted list is loopback and private networks, where a container proxy usually lives; if the app itself is reachable from a private network, narrow it to the proxy's exact address. Cookies are marked Secure when the trusted proxy says the request came over HTTPS. For a proxy that discovers services by container labels, add its labels and network to your `docker-compose.yml`. Proxy-level rate limits (for example nginx `limit_req`) are welcome as a second layer.

### Operations
- **Update**: `docker compose build && docker compose up -d` with the same `-f` files you started with. The health check settles within seconds; schema changes are applied at startup; missing embeddings are filled in the background.
- **Back up**: three places hold everything.
  - The **database** (`db_data` volume, or `pg_dump` of the `studio` database): the model and its versions, the Library's collections, sources, passages, topics and statements, users, tokens, connectors, settings, activity, and the demo database.
  - The **original files** of the Library: `./data/library` with the local driver, or the bucket (`s3_data` volume of the storage overlay, or your S3 service's own backups) with the s3 driver.
  - `./data` otherwise: Excel backups and connector files. Excel exports of the model and zip exports of collections are portable second copies.
- **Restore**: restore the database dump into a new volume, put the files back (data folder or bucket) and start the stack.
- **Scale the work**: add the worker overlay and `--scale worker=N`; the web app stays responsive while documents are processed.
- **Rotate a token**: create a new one, update the agent, revoke the old one.
- **Rotate `SECRET_KEY`**: re-enter connector passwords, the AI key, the S3 secret and the fetcher token if they were set on the page.
- **Pinned images**: the compose files pin exact image versions (PostgreSQL with pgvector, SeaweedFS, the AWS CLI used once to create the bucket, Playwright); bump them deliberately.
- **Vulnerability scan**: `scripts/audit.sh` checks the locked Python packages and the frontend's runtime packages.


## Security
- The app runs as a non-root user and connects to PostgreSQL with its own non-superuser account; the superuser is only used to create it.
- Connections to your data are read-only at the session level, with statement timeouts and row limits. Use read-only database accounts: the connector test warns when an account can write.
- Every data query is checked and regenerated by the server; restricted columns never leave the database without an explicit grant; masked columns leave only as hashes.
- Secrets at rest: connector passwords and an OpenRouter key set on the Settings page are encrypted; session and API tokens are stored hashed; user passwords use scrypt.
- Sessions are HttpOnly, SameSite=Lax cookies, Secure over HTTPS. Responses carry a Content-Security-Policy and the usual hardening headers.
- Rate limits per client address (API and MCP), per address for sign-in, and per user for Alfred; sign-in lockout after repeated failures.
- Connectors cannot reach the app's own database or link-local and cloud-metadata addresses; file databases must live under `CONNECTOR_FILES_DIR`.
- Library: every document-derived row belongs to one collection and is reached only through a scope object that checks grants (no grant is a 404); uploaded files are stored under generated keys, path-checked; web capture refuses private, loopback and metadata addresses, pins the resolved address and re-checks redirects; the optional browser runs in its own container with no database or files; the UI renders model-written Markdown without HTML or images; heavy and AI-calling actions are rate limited per user; document text is treated as data in prompts, and an agent only receives what its grants allow.
- Python dependencies are installed from a lock file with hashes; base images are pinned to exact versions.

## Architecture
```
  people ──► web UI ─┐
                     ├──► FastAPI app ──► PostgreSQL (model, versions, search index, users, audit, activity, settings; schema `library`: collections, sources and their chunks, source wikis and their chunks, topics and their chunks, claims, links, jobs)
  agents ──► /mcp ───┘        │
                              ├──► embedding model (in the container)
                              ├──► job workers (in the app, or the optional worker container)
                              ├──► file storage (data folder, or the S3 bucket: optional container or your own)
                              ├──► browser service (optional container, isolated, for JavaScript pages)
                              ├──► chat model (OpenRouter, for Alfred)
                              └──► your database (connector, read-only) or the built-in demo database
```
One PostgreSQL database with pgvector holds the model, its versions, the search index and the app's own data; no separate graph or vector store is needed at this scale (thousands of chunks, an exact vector scan; tens of entities and hundreds of joins, an in-memory graph). The model is stored as rows per version (entity, field, relationship, example); the Library lives in its own schema, `library`, in the same database; search chunks are derived from it and never edited. The query planner and the join-path finder are deterministic code. Database-specific behaviour (connecting, structure, dialect, masking expression, forbidden functions) lives in one driver class per database kind; SQL is parsed and generated with sqlglot. The UI is React, served by the app.

| Area | Code (`backend/studio/`) |
|---|---|
| Model storage, versions, editing | `ontology.py`, `versions.py`, `editing.py`, `excel_io.py`, `modelview.py` |
| Search | `chunker.py`, `indexer.py`, `embeddings.py`, `search.py` |
| Tools, planner, MCP, Alfred | `tools.py`, `planner.py`, `graph.py`, `filtermatch.py`, `mcp_server.py`, `alfred.py`, `llm.py` |
| Data access | `connectors.py`, `drivers.py`, `datastore.py`, `enforce.py`, `reverse.py` |
| Model check, demo data | `conformance.py`, `databuild.py`, `synth/` (`generic.py` generates, `ddl.py` derives tables, `load.py` loads, `export.py` copies) |
| Users, access, limits, settings, activity | `auth.py`, `access.py`, `clientinfo.py`, `ratelimit.py`, `settings.py`, `activity.py` |
| API, configuration, schema | `api.py`, `config.py`, `db.py`, `schema.sql` |
| Library (documents, topics, bridge) | `library/`: `api.py`, `scope.py` (access), `jobs.py`, `pipeline.py`, `wiki.py`, `topics.py`, `bridge.py`, `agent.py`, `retrieval.py`, `interview.py`, `export.py`, `fetch.py`, `netguard.py`, `storage.py`, `schema.sql` (schema `library`); `worker.py` and `fetcher.py` (optional browser service) in `studio/` |

**App tables** (`schema.sql`, applied at startup): `ontology_version`, `entity`, `field`, `relationship`, `example`, `meta` (the model per version); `chunk`, `embedding_cache` (search); `audit_log`; `app_user`, `session`, `api_token`, `login_attempt` (access); `question_log`; `data_build` (demo database builds); `connector`; `app_setting` (Settings page values); `activity_session`, `activity_event`. The Library's tables are in the `library` schema (`library/schema.sql`): collections and grants, sources, jobs, passages, source-page versions and checks, topics with versions, aliases, links and sources, statements with their evidence, the vocabulary and its proposals, conflicts, redirects, health runs, model links and interviews.

**HTTP API**: the UI's own API under `/api` (signed-in cookie; admin routes under `/api/admin`), the MCP endpoint at `/mcp` (bearer token), `/api/health`. The Excel template and exports are `/api/template.xlsx` and `/api/export.xlsx?version=published|draft`.


## Development
```sh
python -m venv .venv && .venv/bin/pip install --require-hashes -r backend/requirements.lock && .venv/bin/pip install -r backend/requirements-dev.txt   # test tools (pytest, moto)
docker run -d --name seamory-testdb -e POSTGRES_PASSWORD=dev -p 127.0.0.1:55432:5432 pgvector/pgvector:0.8.7-pg16-bookworm
export TEST_DATABASE_URL=postgresql://postgres:dev@127.0.0.1:55432/postgres    # any PostgreSQL superuser on a server with pgvector
cd backend && ../.venv/bin/python -m pytest -q tests      # creates and drops its own throwaway databases; never calls an AI model or the network
cd frontend && npm ci && npx vite build                   # the UI (React)
```
CI (`.github/workflows/ci.yml`) runs the suite at two embedding sizes (768, and 3000 for the halfvec path), builds the UI and both images, and audits the dependencies. See [CONTRIBUTING.md](CONTRIBUTING.md) and [SECURITY.md](SECURITY.md).
The tests run on a fictional model of a subscription business, `backend/tests/fixtures/sample_model.py`, read through the product's own Excel format; it covers every concept the ontology supports (supertypes, hierarchies, optional and unreliable links, masked and restricted fields, filters, checks, calculated fields, examples). `backend/devtools/` has two checks that use it: `python -m devtools.evalcli` (search quality on `tests/retrieval_cases.yaml`) and `python -m devtools.agenteval` (Alfred's answers to multi-entity questions against hand-written SQL; calls the chat model), and `devtools/libraryeval.py` (run inside a running instance's container: builds two throw-away collections, one public and one internal, from fictional documents, asks Alfred seven questions including a customer reply that must not quote the internal playbook, and deletes them; calls the chat model).

Dependencies: `backend/requirements.in` lists the direct ones; `requirements.lock` (exact versions with hashes) is generated with `pip-compile --generate-hashes --allow-unsafe -o requirements.lock requirements.in`. Run `scripts/audit.sh` after any change.


## Limits
- One model per installation, one active connector at a time.
- Row-level security (rows filtered per caller) is not implemented; access is decided per column.
- Agents receive at most `AGENT_MAX_ROWS` rows per query; results are not paged (they are told the total and to aggregate or filter).
- Rate limits are kept in memory: one app process. Several app instances need a shared store.
- Sign-in is username and password; single sign-on and multi-factor authentication are not built in yet.
- Translation of the model's SQL to other databases is best effort; constructs that do not translate are reported by the Model check.
- The demo database is generic: it follows the model's keys, links and allowed values, not your business rules.
- Library: collections are isolated by grants, not by separate databases; one embedding model and dimension per installation (changing it means re-embedding); tokens get read grants only; scanned pages are read by a model, up to `OCR_MAX_PAGES` per document; the browser service does not fetch pages behind a login; collection export has no matching import yet; changing the storage driver does not move files already stored.
- State-changing browser requests rely on the session cookie's SameSite setting; there is no separate CSRF token.
