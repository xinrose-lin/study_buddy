# Product Requirements Document
## Smart Learning Companion
**Version:** 2.0
**Author:** Lin Xin Rose
**Last updated:** June 2026
**Goal:** Clean, lightweight, open source tool for self-directed learners to capture notes, build tag connections, and get AI help as they learn.

---

## 1. Overview

A Chrome extension + AWS serverless backend + hosted MCP server that lets a user:
- Capture highlights and freeform notes from any webpage or PDF
- Tag notes to topics and define relationships between topics
- Persist everything to AWS (their own account — self-hosted)
- Query notes conversationally via Claude.ai using a hosted MCP server
- Visualise tag relationships as an interactive D3.js force-directed graph
- Get on-demand AI analysis via AWS Bedrock

**Open source principles:**
- Zero vendor lock-in on AI provider (Bedrock swappable via config)
- Self-hostable: each user deploys to their own AWS account
- One-command deploy via SAM
- Minimal dependencies: vanilla JS extension, Python Lambda, D3.js graph only
- Clean interfaces so contributors can extend without breaking things

---

## 2. Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Chrome extension | Vanilla JS, Manifest V3 | No framework overhead, easy to contribute to |
| PDF rendering | PDF.js (bundled) | Full text layer control |
| IaC / deploy | AWS SAM (YAML) | Readable, teaches CloudFormation fundamentals |
| Backend compute | AWS Lambda (Python 3.12) | Serverless, low cost, self-hostable |
| API | AWS API Gateway (REST) | Pairs with Lambda, API key auth built in |
| Database | AWS DynamoDB | Fast tag-based queries, pay-per-request |
| AI — basic | AWS Bedrock, Amazon Titan | Cheap, fast, summarisation |
| AI — quality | AWS Bedrock, Claude Sonnet | Higher quality analysis |
| Graph viz | D3.js (force-directed) | Lightweight (~30kb), tree-shakeable, industry standard |
| MCP server | Python `mcp` lib, Lambda | Hosted, always-on, Claude.ai compatible |
| Logging | AWS CloudWatch | Structured logs, industry standard |
| Alerts | AWS SNS | Ops practice |
| Future storage | AWS S3 | Long-form notes >2000 chars |
| Future auth | AWS Cognito | Multi-user sharing |

---

## 3. Architecture

```
Chrome Extension (Manifest V3 + PDF.js + D3.js)
        │
        ▼  REST + x-api-key header
API Gateway
        │
        ▼
Lambda (Python 3.12)
  ├── notes_handler.py     — CRUD
  ├── ai_handler.py        — Bedrock calls
  ├── mcp_handler.py       — MCP protocol
  └── utils/
      ├── db.py            — DynamoDB helpers
      ├── dedup.py         — hash-based dedup
      └── logger.py        — structured CloudWatch logs
        │
   ┌────┼──────────┐
   ▼    ▼          ▼
DynamoDB  Bedrock  CloudWatch
                       │
                      SNS (alerts, Phase 4)

Claude.ai ──MCP HTTPS──▶ API Gateway ──▶ mcp_handler.py ──▶ DynamoDB
```

---

## 4. Data Schema

### DynamoDB — Notes table

```json
{
  "note_id":      "uuid-v4 (partition key) — stable forever, never changes even if content is edited",
  "user_id":      "default (hardcoded MVP; supports future multi-user)",
  "content":      "string, max 2000 chars",
  "tags":         "DynamoDB StringSet (SS type) — supports contains() filter query",
  "source_type":  "web | pdf | freeform",
  "source_url":   "string or null",
  "source_title": "string or null",
  "source_page":  "integer or null (PDF only)",
  "created_at":   "ISO 8601 — set once on create, never updated",
  "updated_at":   "ISO 8601 — set on create, updated on every edit",
  "hash":         "sha256(content + source_url) — recalculated on every edit, used for dedup on create"
}
```

### DynamoDB — Tags table

```json
{
  "tag_id":       "string (partition key)",
  "related_tags": [
    {"tag": "infra", "relationship": "subset_of", "ai_suggested": false}
  ],
  "note_count":   12,
  "created_at":   "ISO 8601"
}
```

### DynamoDB indexes

| Index | Type | On | Enables |
|---|---|---|---|
| `hash-index` | GSI | `hash` | Dedup check on create |

**No `tags-index` for MVP.** Tags stored as DynamoDB `StringSet (SS)` and queried via `scan()` with `Attr('tags').contains('kubernetes')`. Fast enough for a personal tool (<1000 notes). Add a proper tags index if multi-user scale requires it later.

---

## 5. API Endpoints

All require `x-api-key` header.

| Method | Path | Handler | Description |
|---|---|---|---|
| POST | `/notes` | notes_handler | Save note (with dedup) |
| GET | `/notes` | notes_handler | List notes (`?tag=` optional) |
| GET | `/notes/{id}` | notes_handler | Get single note |
| DELETE | `/notes/{id}` | notes_handler | Delete note |
| PUT | `/notes/{id}` | notes_handler | Edit note content and/or tags (recalculates hash, updates updated_at) |
| GET | `/topics` | notes_handler | List all tags + counts |
| PUT | `/topics/{id}/relations` | notes_handler | Add/update tag relationship |
| POST | `/ai/summarise` | ai_handler | Summarise topic notes |
| POST | `/ai/connections` | ai_handler | Suggest tag relationships |
| POST | `/ai/quiz` | ai_handler | Generate quiz questions |
| POST | `/mcp` | mcp_handler | MCP protocol endpoint |

---

## 6. MCP Tools

| Tool | Args | Description |
|---|---|---|
| `get_notes_by_topic` | `topic: str` | All notes for a tag |
| `search_notes` | `query: str` | Keyword search |
| `add_note` | `content: str, tags: list` | Save from Claude |
| `list_topics` | none | All tags + counts |
| `get_tag_graph` | none | Nodes + edges for graph |

---

## 7. Open Source Setup Requirements

Every contributor / new user needs to be able to:

```bash
git clone https://github.com/roselin/learning-companion
cd learning-companion
cp .env.example .env        # fill in AWS region + API key name
sam build
sam deploy --guided         # creates all AWS resources
# Load extension in Chrome: chrome://extensions → Load unpacked → /extension
```

`.env.example` must include:
```
AWS_REGION=ap-southeast-1
API_KEY_NAME=learning-companion-key
DYNAMODB_NOTES_TABLE=notes
DYNAMODB_TAGS_TABLE=tags
BEDROCK_BASIC_MODEL=amazon.titan-text-lite-v1
BEDROCK_QUALITY_MODEL=anthropic.claude-sonnet-4-5
```

---

## 8. Phased Delivery — Detailed

---

### PHASE 0 — AWS Foundation
*Goal: deploy the skeleton backend, verify it works end to end before writing any extension code.*

#### Step 0.1 — Project scaffold
- [ ] Create repo structure:
  ```
  learning-companion/
  ├── backend/
  │   ├── handlers/
  │   │   ├── notes_handler.py
  │   │   ├── ai_handler.py
  │   │   └── mcp_handler.py
  │   ├── utils/
  │   │   ├── db.py
  │   │   ├── dedup.py
  │   │   └── logger.py
  │   └── requirements.txt
  ├── extension/
  │   ├── manifest.json
  │   ├── popup/
  │   ├── content/
  │   └── background/
  ├── template.yaml          — SAM template
  ├── .env.example
  └── README.md
  ```
- [ ] `requirements.txt`: `boto3`, `mcp`

**How to test manually:**
```bash
# verify project structure looks right
tree learning-companion/
```

---

#### Step 0.2 — SAM template: DynamoDB tables
- [ ] Define `NotesTable` in `template.yaml` with partition key `note_id`
- [ ] Define `TagsTable` with partition key `tag_id`
- [ ] Add GSI `hash-index` on `NotesTable` (only GSI needed for MVP)
- [ ] Set `BillingMode: PAY_PER_REQUEST` (no provisioned capacity costs)
- [ ] Confirm Notes table has both `created_at` (immutable) and `updated_at` (mutable) fields documented in schema comments

**How to test manually:**
```bash
sam build
sam deploy --guided
# then in AWS Console → DynamoDB → Tables
# verify both tables exist with correct indexes
# Console: https://console.aws.amazon.com/dynamodb
```

---

#### Step 0.3 — SAM template: Lambda + API Gateway
- [ ] Define `NotesFunction` Lambda pointing to `notes_handler.lambda_handler`
- [ ] Add API Gateway events: POST/GET `/notes`, GET/DELETE `/notes/{id}`, GET `/topics`
- [ ] Add `DynamoDBCrudPolicy` for both tables
- [ ] Define API key + usage plan in SAM template
- [ ] Add `LOG_LEVEL`, `NOTES_TABLE`, `TAGS_TABLE` env vars to Lambda

**How to test manually:**
```bash
sam deploy
# AWS Console → API Gateway → your API → API Keys
# copy the key value

# test with curl:
curl -X POST https://YOUR_API_ID.execute-api.ap-southeast-1.amazonaws.com/Prod/notes \
  -H "x-api-key: YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"content": "test note", "tags": ["test"], "source_type": "freeform"}'

# expect: {"note_id": "...", "created_at": "..."}
```

---

#### Step 0.4 — Structured logger
- [ ] Write `utils/logger.py` — wraps Python `logging`, outputs JSON to CloudWatch:
  ```json
  {"event": "note_saved", "note_id": "...", "duration_ms": 42, "error": null}
  ```
- [ ] Import and use in all handlers from day one

**How to test manually:**
```bash
# after invoking any endpoint, go to:
# AWS Console → CloudWatch → Log groups → /aws/lambda/NotesFunction
# verify log lines are valid JSON
```

---

#### Step 0.5 — Dedup utility
- [ ] Write `utils/dedup.py`:
  - `compute_hash(content, source_url)` → `sha256(content + (source_url or ""))`
  - `check_duplicate(hash)` → query `hash-index` GSI, return existing note or None
  - `should_check_dedup(note_id)` → True if creating (note_id is None), False if editing
- [ ] Unit test locally with `pytest`

**How to test manually:**
```bash
# save the same note twice:
curl -X POST .../notes -d '{"content": "hello", "tags": ["test"], "source_type": "freeform"}'
curl -X POST .../notes -d '{"content": "hello", "tags": ["test"], "source_type": "freeform"}'

# second call should return: {"duplicate": true, "note_id": "...existing id..."}
# verify only ONE item in DynamoDB Console → Tables → notes → Explore items
```

---

### PHASE 1 — Notes CRUD API
*Goal: all five note endpoints working and manually verified.*

#### Step 1.1 — POST /notes
- [ ] Validate request body (content required, max 2000 chars, tags is a list)
- [ ] Run dedup check
- [ ] Write to DynamoDB
- [ ] Return created note with 201 status
- [ ] Log `note_saved` event

**How to test manually:**
```bash
# save a real note
curl -X POST .../notes \
  -H "x-api-key: KEY" \
  -d '{"content": "Kubernetes pods are the smallest deployable unit", "tags": ["kubernetes", "infra"], "source_type": "web", "source_url": "https://kubernetes.io/docs", "source_title": "Kubernetes docs"}'

# verify in DynamoDB Console → Tables → notes → Explore items
# verify in CloudWatch → log group → latest stream
```

---

#### Step 1.2 — GET /notes and GET /notes/{id}
- [ ] `GET /notes` returns all notes for `user_id = default`, sorted by `created_at` desc
- [ ] `GET /notes?tag=kubernetes` uses `scan()` with `Attr('tags').contains('kubernetes')` — StringSet query, no GSI needed
- [ ] `GET /notes/{id}` returns single note or 404

**How to test manually:**
```bash
# list all
curl .../notes -H "x-api-key: KEY"

# filter by tag
curl ".../notes?tag=kubernetes" -H "x-api-key: KEY"

# get by id (use id from POST response)
curl .../notes/YOUR_NOTE_ID -H "x-api-key: KEY"

# bad id — expect 404
curl .../notes/does-not-exist -H "x-api-key: KEY"
```

---

#### Step 1.3 — DELETE /notes/{id}
- [ ] Delete item from DynamoDB
- [ ] Return 204 on success, 404 if not found
- [ ] Log `note_deleted` event

**How to test manually:**
```bash
# delete a note
curl -X DELETE .../notes/YOUR_NOTE_ID -H "x-api-key: KEY"
# expect 204

# try to get it — expect 404
curl .../notes/YOUR_NOTE_ID -H "x-api-key: KEY"
```

---

#### Step 1.4 — PUT /notes/{id} (edit note)
- [ ] Accept `content` and/or `tags` in request body (both optional)
- [ ] If `content` changed → recalculate hash, update `hash` field
- [ ] Always update `updated_at` to current timestamp
- [ ] `note_id` and `created_at` never change
- [ ] Skip dedup check on edit (user is intentionally changing content)
- [ ] Return updated note

**How to test manually:**
```bash
# save a note first, get its note_id
NOTE_ID=$(curl -s -X POST .../notes   -H "x-api-key: KEY"   -d '{"content": "original content", "tags": ["test"], "source_type": "freeform"}'   | python -m json.tool | grep note_id | cut -d'"' -f4)

# edit the note
curl -X PUT .../notes/$NOTE_ID   -H "x-api-key: KEY"   -d '{"content": "edited content", "tags": ["test", "edited"]}'

# verify note_id is unchanged, content updated, updated_at is newer than created_at
curl .../notes/$NOTE_ID -H "x-api-key: KEY"

# verify dedup: save the original content again → should create NEW note (different hash now)
curl -X POST .../notes   -H "x-api-key: KEY"   -d '{"content": "original content", "tags": ["test"], "source_type": "freeform"}'
# expect: new note_id (not duplicate), because edited note now has different hash
```

---

#### Step 1.5 — GET /topics and PUT /topics/{id}/relations
- [ ] `GET /topics` scans tags table, returns `[{tag_id, note_count, related_tags}]`
- [ ] `PUT /topics/{id}/relations` upserts a relationship:
  ```json
  {"related_tag": "infra", "relationship": "subset_of", "ai_suggested": false}
  ```
- [ ] Auto-create tag record if it doesn't exist yet

**How to test manually:**
```bash
# list topics (after saving some notes)
curl .../topics -H "x-api-key: KEY"

# add a manual relationship
curl -X PUT .../topics/kubernetes/relations \
  -H "x-api-key: KEY" \
  -d '{"related_tag": "infra", "relationship": "subset_of", "ai_suggested": false}'

# list topics again — verify relationship appears
curl .../topics -H "x-api-key: KEY"
```

---

### PHASE 2 — Chrome Extension MVP
*Goal: working popup that saves notes to your real AWS backend.*

#### Step 2.1 — Manifest and project setup
- [ ] `manifest.json` Manifest V3 with:
  - `permissions`: `storage`, `activeTab`, `contextMenus`, `scripting`
  - `host_permissions`: your API Gateway URL
  - `action`: popup
  - `background`: service worker
- [ ] `.env`-style config in `extension/config.js`:
  ```js
  export const API_URL = 'https://YOUR_ID.execute-api...';
  export const API_KEY = 'YOUR_KEY';
  ```

**How to test manually:**
```
Chrome → chrome://extensions → Enable Developer mode
→ Load unpacked → select /extension folder
→ extension icon should appear in toolbar
→ click it → popup should open (even if empty)
```

---

#### Step 2.2 — Popup UI: browse tab
- [ ] Topic list view: fetch `GET /topics`, render tag pills with note counts
- [ ] Click tag → fetch `GET /notes?tag=X` → render note cards
- [ ] Note card shows: content preview (100 chars), source title, date, delete button
- [ ] Delete button calls `DELETE /notes/{id}`, refreshes list

**How to test manually:**
```
Open popup → should see your test topics from Phase 1
Click a topic → should see notes you saved via curl
Click delete → note should disappear
Refresh popup → deleted note should still be gone
```

---

#### Step 2.3 — Freeform note capture
- [ ] "New note" tab in popup with textarea + tag input + char counter
- [ ] Tag input: type tag, press Enter or comma to add, show as removable pills
- [ ] Save button → `POST /notes` with `source_type: freeform`
- [ ] Success: show green toast, clear form, switch to browse tab

**How to test manually:**
```
Open popup → New note tab
Type a note → add 2 tags → click Save
Switch to Browse tab → note should appear under correct tags
Open DynamoDB Console → verify item exists with correct fields
```

---

#### Step 2.4 — Web highlight capture
- [ ] Content script injected on all pages
- [ ] On text selection → show floating "Save" button near selection
- [ ] Click Save → open small overlay with: selected text (editable), tag input, source auto-filled
- [ ] Submit → `POST /notes` with `source_type: web`, `source_url`, `source_title`

**How to test manually:**
```
Go to any article (e.g. a Kubernetes docs page)
Select a sentence → floating Save button should appear
Click it → overlay appears with your text and the page URL pre-filled
Add tags → submit
Open popup → note should appear
Check DynamoDB → source_url should be the page you were on
```

---

#### Step 2.5 — Obsidian export
- [ ] Export button in popup footer
- [ ] Options: all notes / by selected tag
- [ ] Generates and downloads `.md` file:
  ```markdown
  # kubernetes
  
  ## Note — 2026-06-01
  Source: [Kubernetes docs](https://kubernetes.io)
  Tags: kubernetes, infra
  
  Kubernetes pods are the smallest deployable unit.
  
  ---
  ```

**How to test manually:**
```
Save 3+ notes across 2 topics
Click Export → select a topic → file downloads
Open file in any text editor — check formatting
Drag file into Obsidian vault → should render as linked note
```

---

### PHASE 3 — PDF Support
*Goal: highlight text in PDFs rendered in the browser.*

#### Step 3.1 — Bundle PDF.js
- [ ] Add PDF.js to `extension/lib/pdfjs/`
- [ ] Add `pdf_viewer.html` as an extension page
- [ ] Register context menu: "Open with Learning Companion" on PDF links (background service worker)

**How to test manually:**
```
Right-click any PDF link on the web
"Open with Learning Companion" should appear in context menu
Click it → PDF.js viewer should open in a new extension tab
```

---

#### Step 3.2 — PDF highlight capture
- [ ] Text layer enabled in PDF.js viewer
- [ ] On text selection in PDF viewer → show Save overlay (same component as web)
- [ ] Source metadata: `source_type: pdf`, `source_title: filename`, `source_page: pageNumber`

**How to test manually:**
```
Open a PDF via the extension viewer (try a research paper PDF)
Select text on page 3 → Save overlay appears
Add tags → submit
Open popup → note appears
Check DynamoDB → source_page should be 3, source_type should be "pdf"
```

---

### PHASE 4 — D3.js Tag Graph
*Goal: interactive visual of how your topics connect.*

#### Step 4.1 — Graph data endpoint
- [ ] `GET /topics` already returns `related_tags` — confirm shape matches D3 needs:
  ```json
  {
    "nodes": [{"id": "kubernetes", "note_count": 12}],
    "edges": [{"source": "kubernetes", "target": "infra", "relationship": "subset_of"}]
  }
  ```
- [ ] Add `/graph` endpoint that formats topics data for D3

**How to test manually:**
```bash
curl .../graph -H "x-api-key: KEY"
# verify nodes array has all your tags
# verify edges array reflects relationships you defined in Phase 1.4
```

---

#### Step 4.2 — D3 force-directed graph
- [ ] `extension/graph/graph.html` — full-page graph view
- [ ] D3 imports: `d3-force`, `d3-selection`, `d3-drag`, `d3-zoom` only (keep bundle small)
- [ ] Nodes: circles sized by `note_count`, labeled with tag name
- [ ] Edges: lines labeled with relationship type (`subset_of`, `relates_to`)
- [ ] Forces: `forceManyBody` (repulsion), `forceLink` (edge tension), `forceCenter`
- [ ] Interactions: drag nodes, scroll to zoom, click node → filter popup to that topic

**How to test manually:**
```
Open popup → click "Tag graph" button → graph.html opens
Should see your topics as circles, connected by lines
Drag a node → it should move, others should react (force simulation)
Scroll → should zoom in/out
Click a node → popup notes list should filter to that tag
```

---

#### Step 4.3 — Manual relationship editor in graph
- [ ] Right-click node → context menu: "Add relationship"
- [ ] Small form: select target tag, select relationship type, save
- [ ] Calls `PUT /topics/{id}/relations`
- [ ] Graph re-renders with new edge

**How to test manually:**
```
In graph view, right-click "ansible" node
Select "Add relationship" → choose "infra", type "relates_to" → save
New edge should appear between ansible and infra immediately
Refresh graph → edge should persist (stored in DynamoDB)
```

---

### PHASE 5 — AI Analysis (Bedrock)
*Goal: on-demand AI summaries, connection suggestions, quiz mode.*

#### Step 5.1 — Bedrock setup and ai_handler.py
- [ ] Add `BedrockFullAccess` policy to Lambda in SAM template
- [ ] Write `ai_handler.py` with Bedrock client:
  ```python
  import boto3
  bedrock = boto3.client('bedrock-runtime', region_name=os.environ['AWS_REGION'])
  ```
- [ ] Helper: `call_model(prompt, model_id)` — handles both Titan and Sonnet
- [ ] Route: `POST /ai/summarise`, `POST /ai/connections`, `POST /ai/quiz`

**How to test manually:**
```bash
# summarise a topic
curl -X POST .../ai/summarise \
  -H "x-api-key: KEY" \
  -d '{"topic": "kubernetes", "model": "basic"}'
# expect: {"summary": "...3-5 sentences..."}

# try quality model
curl -X POST .../ai/summarise \
  -d '{"topic": "kubernetes", "model": "quality"}'
# compare output — quality should be noticeably better
```

---

#### Step 5.2 — AI connection suggestions
- [ ] `POST /ai/connections` sends all tag names + sample note content to Bedrock
- [ ] Prompt asks for JSON array of suggested relationships
- [ ] Response parsed and returned:
  ```json
  [{"from": "ansible", "to": "automation", "relationship": "relates_to", "confidence": 0.9}]
  ```
- [ ] Extension: "AI suggestions" button in graph view → shows suggestions as dashed edges
- [ ] User can accept (saves to DynamoDB) or dismiss each

**How to test manually:**
```bash
curl -X POST .../ai/connections -H "x-api-key: KEY"
# verify response is valid JSON array
# verify confidence scores are present

# in extension graph view:
# click "AI suggestions" → dashed edges appear
# click Accept on one → edge becomes solid, persists on refresh
# click Dismiss → edge disappears
```

---

#### Step 5.3 — Quiz mode
- [ ] `POST /ai/quiz` with `{"topic": "kubernetes"}`
- [ ] Returns 3 questions:
  ```json
  [{"question": "What is a pod?", "answer": "The smallest deployable unit in Kubernetes"}]
  ```
- [ ] Extension popup: "Quiz me" button per topic → renders Q&A cards, reveal answer on click

**How to test manually:**
```bash
curl -X POST .../ai/quiz \
  -H "x-api-key: KEY" \
  -d '{"topic": "kubernetes"}'
# verify 3 questions returned
# verify answers are grounded in your actual notes (not generic Kubernetes facts)

# in extension:
# click Quiz me → 3 question cards appear, answers hidden
# click a card → answer reveals
```

---

### PHASE 6 — MCP Server
*Goal: Claude.ai can query your notes conversationally.*

#### Step 6.1 — mcp_handler.py
- [ ] Implement MCP protocol over HTTP POST:
  - `tools/list` → return all 5 tool schemas
  - `tools/call` → route to correct tool function
- [ ] Tools call same `db.py` helpers as notes_handler
- [ ] Add `/mcp` route to SAM template (same Lambda, different path)

**How to test manually:**
```bash
# list tools
curl -X POST .../mcp \
  -H "x-api-key: KEY" \
  -d '{"method": "tools/list"}'
# expect: {"tools": [...5 tool schemas...]}

# call a tool
curl -X POST .../mcp \
  -H "x-api-key: KEY" \
  -d '{"method": "tools/call", "params": {"name": "list_topics", "arguments": {}}}'
# expect: {"content": [{"type": "text", "text": "[\"kubernetes\", \"infra\", ...]"}]}
```

---

#### Step 6.2 — Connect to Claude.ai
- [ ] Note your MCP endpoint URL from SAM deploy output
- [ ] In Claude.ai: Settings → Integrations → Add MCP server → paste URL + API key

**How to test manually:**
```
In Claude.ai, type: "What topics have I been learning about?"
Claude should call list_topics → respond with your actual tags

Type: "Summarise my notes on kubernetes"
Claude should call get_notes_by_topic("kubernetes") → summarise your real notes

Type: "Save a note: DynamoDB uses partition keys for horizontal scaling. Tag it as dynamodb and infra"
Claude should call add_note → verify in DynamoDB Console the note was created
```

---

### PHASE 7 — Observability
*Goal: production-grade logging and alerting.*

#### Step 7.1 — Structured CloudWatch logs
- [ ] Ensure all handlers emit structured JSON logs for every operation
- [ ] Add `duration_ms` to every log line (measure with `time.time()`)
- [ ] Log errors with full traceback under `"error"` key

**How to test manually:**
```
AWS Console → CloudWatch → Log groups → /aws/lambda/NotesFunction
Invoke a few endpoints
Verify every log line is valid JSON
Verify error logs appear when you send a bad request (e.g. missing content field)
```

---

#### Step 7.2 — CloudWatch dashboard
- [ ] Create dashboard via SAM (CloudFormation `AWS::CloudWatch::Dashboard`):
  - Notes saved per day (metric filter on `event = note_saved`)
  - Lambda error rate
  - API Gateway 4xx/5xx count
  - Bedrock query latency (p50, p95)

**How to test manually:**
```
AWS Console → CloudWatch → Dashboards → LearningCompanion
Save 5 notes, call AI endpoint twice
Refresh dashboard → metrics should update (may take 1-2 min)
```

---

#### Step 7.3 — SNS alert
- [ ] Add `AWS::SNS::Topic` and `AWS::CloudWatch::Alarm` to SAM template
- [ ] Alarm: Lambda error rate > 5% over 5-minute window → email via SNS
- [ ] Subscribe your email to the SNS topic

**How to test manually:**
```bash
# deliberately trigger errors by sending malformed requests in a loop:
for i in {1..10}; do
  curl -X POST .../notes -H "x-api-key: KEY" -d '{"bad": "payload"}'
done

# wait 5-10 minutes → check your email for SNS alert
# AWS Console → CloudWatch → Alarms → verify alarm state changed to ALARM
```

---

### PHASE 8 — Open Source Polish
*Goal: someone can clone and deploy in under 30 minutes.*

#### Step 8.1 — README
- [ ] Prerequisites (AWS CLI, SAM CLI, Python 3.12, Chrome)
- [ ] 5-step quickstart: clone → `.env` → `sam deploy` → load extension → test with curl
- [ ] Architecture diagram (link to the one in this PRD)
- [ ] How to contribute (fork, branch, PR)
- [ ] How to swap AI provider (point to `ai_handler.py` config)

#### Step 8.2 — Config externalisation
- [ ] All hardcoded values moved to `.env` / Lambda env vars
- [ ] `config.js` in extension reads from build-time constants
- [ ] `user_id = "default"` clearly commented as MVP placeholder

#### Step 8.3 — GitHub Actions CI
- [ ] `pytest` runs on every PR (unit tests for `dedup.py`, `db.py`, handlers)
- [ ] `sam validate` runs on every PR (catches template errors)

**How to test manually:**
```bash
# run tests locally before pushing
cd backend && pytest tests/ -v

# validate SAM template
sam validate

# simulate a fresh install — delete all your AWS resources:
sam delete
# then redeploy from scratch using only the README instructions
# if it works cleanly, the README is good
```

---

## 9. Testing Strategy Summary

| Phase | What to test | How |
|---|---|---|
| 0 | AWS resources created | DynamoDB Console, curl |
| 1 | All 5 CRUD endpoints | curl commands |
| 2 | Extension UI, note save flow | Manual browser testing |
| 3 | PDF highlight capture | Open PDF, highlight, check DynamoDB |
| 4 | D3 graph renders + interactions | Browser, drag/click nodes |
| 5 | AI responses grounded in your notes | curl + eyeball |
| 6 | MCP tools work in Claude.ai | Chat with Claude |
| 7 | Logs appear in CloudWatch | Console + intentional errors |
| 8 | Fresh install works | Delete stack, redeploy |

### Useful curl aliases (add to your .bashrc)
```bash
export LC_API="https://YOUR_ID.execute-api.ap-southeast-1.amazonaws.com/Prod"
export LC_KEY="YOUR_API_KEY"

alias lc-notes="curl -s $LC_API/notes -H 'x-api-key: $LC_KEY' | python -m json.tool"
alias lc-topics="curl -s $LC_API/topics -H 'x-api-key: $LC_KEY' | python -m json.tool"
alias lc-save='f(){ curl -s -X POST $LC_API/notes -H "x-api-key: $LC_KEY" -H "Content-Type: application/json" -d "$1" | python -m json.tool; }; f'

# usage:
lc-topics
lc-save '{"content": "my note", "tags": ["test"], "source_type": "freeform"}'
```

---

## 10. Open Questions

- [ ] Tag graph: node colour scheme — by topic category or note count intensity?
- [ ] AI provider swap: define a clean interface so contributors can add OpenAI/Gemini support
- [ ] SAM vs CDK: revisit at Phase 7 if infra complexity grows

---

## 11. Out of Scope (MVP)

- Multi-user auth (Cognito deferred to post-Phase 8)
- Long-form notes >2000 chars (S3 deferred)
- Continuous Obsidian sync (manual export only)
- Vector/semantic search (keyword scan only)
- Mobile app
- Other browsers