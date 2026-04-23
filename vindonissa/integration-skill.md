
# Vindonissa API — AI Agent Integration Skill

## For the human reading this

You downloaded this file because you want to integrate Helvetii's **Vindonissa** document-processing API into your codebase.

**How to use it:**

1. Open this file in your AI coding assistant (Claude Code, Cursor, Copilot, Windsurf, Aider, or any agent that can read files and write code) from inside your project directory.
2. Tell the agent: *"Follow this skill to wire up Vindonissa."*
3. Answer the questions the agent asks (project_id, API key env var, polling style, etc.).
4. The agent writes the integration code into your repo.

Everything below this line is written **for the AI agent**, not for you. You can stop reading here.

---

## Instructions for the AI agent

You are helping the user integrate the **Vindonissa** document-processing API into their existing codebase. The user has downloaded this skill because they want working code, not documentation. Your job: interview → generate code → verify.

Do not invent fields or endpoints. Only use what is documented in the **API reference** section below. If a question is not answered here, ask the user rather than guess.

### Step 1 — Detect the user's stack

Before asking anything, look at the repository you are in:

- `package.json` → TypeScript / Node.js
- `pyproject.toml` / `requirements.txt` / `setup.py` → Python
- `go.mod` → Go
- `Cargo.toml` → Rust
- `pom.xml` / `build.gradle` → Java / Kotlin
- None of the above → ask the user what language to target.

Match their existing HTTP client (e.g. `axios` if already in `package.json`, `httpx` if already in `pyproject.toml`). Do **not** add a new dependency unless nothing suitable is present.

### Step 2 — Interview the user

Ask these one at a time. Use the session's native question UI if available; otherwise ask in plain text.

1. **API key environment variable name.** Recommend `VINDONISSA_API_KEY`. Remind them:
   - `export VINDONISSA_API_KEY=vnd_xxx` in their shell (or add to `.env` with `.env` in `.gitignore`).
   - Never hardcode the key. Never commit it.
   - The key must start with `vnd_`.
2. **`project_id`** — UUID, paste as-is. (If the user doesn't have it, tell them to get it from their Helvetii contact or the Vindonissa dashboard. Do not try to discover it via the API.)
3. **`document_type_id`** — optional UUID. If they skip it, Vindonissa will auto-classify. Accept either "skip" or a UUID.
4. **Base URL** — default `https://api.helvetii.ai`. Allow override (e.g. staging).
5. **Polling strategy** — offer three presets:
   - **Fixed interval (recommended):** poll every 5s, give up after 5min.
   - **Exponential backoff:** start at 2s, double up to 30s, cap total at 10min.
   - **Submit only:** return the `task_id` and let the caller poll elsewhere.
6. **Output path** — propose a file path based on the repo layout (e.g. `src/integrations/vindonissa.py` or `src/lib/vindonissa.ts`). Confirm with the user before writing.

### Step 3 — Generate the integration

Write **one module** that exposes two functions (names should match language idioms):

- `submit(file_bytes, file_name, project_id, document_type_id=None,  high_precision=False, force_ocr=False) -> task_id`
- `wait_for_result(task_id) -> result_dict`  *(omit if user picked "Submit only")*

Plus **one usage example** that submits a local file and prints the result.

Rules:

- Read the API key from the env var name the user chose. Never take it as a function argument. Fail fast with a clear error if the env var is unset.
- Base64-encode the file bytes before POSTing. Use the language's standard base64 encoder.
- Set `Authorization: Bearer <key>` on every request.
- On `status == "failed"`, raise/throw an error that includes the `error` field from the response. Do **not** retry automatically.
- On network error during polling, retry with the same interval. Do not reset the overall timeout.
- Match the user's existing error-handling and logging style. If the repo uses async, write async code. If it uses sync, write sync code.
- Do **not** add: README changes, CI config, test files (unless the user explicitly asks), new dependencies beyond the HTTP client already in the repo.

### Step 4 — Verify before declaring done

Run this checklist yourself before telling the user you're finished:

1. API key is read from the env var, not hardcoded anywhere.
2. Both endpoints use `Authorization: Bearer <key>`.
3. Polling terminates on `status in ("complete", "failed")`.
4. The code style (imports, naming, async/sync, error handling) matches the surrounding codebase.
5. Print a ready-to-run `curl` equivalent so the user can sanity-check the request shape without executing the generated code.

Then tell the user:

- Which file(s) you wrote.
- The exact shell command to run the example (including the `export` for the env var).
- What a successful result looks like (`status: "complete"`, `result: {...}`).

---

## API reference (ground truth)

**Base URL:** `https://api.helvetii.ai` (production). Staging URL is an AWS ALB hostname — ask the user.

**Auth:** every request requires `Authorization: Bearer vnd_xxxxx`. Keys start with `vnd_`.

### POST /process_file

Submit a file for processing.

**Request body (JSON):**

| Field | Type | Required | Notes |
|---|---|---|---|
| `file_base64` | string | yes | Base64-encoded file bytes |
| `file_name` | string | no | Original filename with extension |
| `project_id` | uuid | yes* | *or* `project` by name — exactly one required |
| `project` | string | yes* | Project name, alternative to `project_id` |
| `document_type_id` | uuid | no | *or* `document_type` by name |
| `document_type` | string | no | Document type name |
| `schema` | object | no | JSON schema to override stored schema for this request |
| `max_pages` | int | no | Default 10. Caps pages processed |
| `high_precision` | bool | no | Default false. Slower, higher-accuracy OCR |
| `force_ocr` | bool | no | Default false. Bypass OCR cache |

**Response:** `{ "task_id": "<uuid>" }`

### GET /task_status/{task_id}

Poll for result.

**Response:**

```json
{
  "task_id": "<uuid>",
  "status": "not_started | in_progress | complete | failed",
  "result": { ... } | null,
  "error": "string or null",
  "created_at": "ISO-8601",
  "updated_at": "ISO-8601"
}
```

**Done condition:** `status` is `complete` (success → read `result`) or `failed` (error → read `error`).

### Example curl flow

```bash
# 1. Submit
curl -X POST https://api.helvetii.ai/process_file \
  -H "Authorization: Bearer $VINDONISSA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "file_base64": "'"$(base64 -i my.pdf)"'",
    "file_name": "my.pdf",
    "project_id": "<your-project-uuid>",
    "max_pages": 10
  }'
# → {"task_id": "abc-123"}

# 2. Poll
curl -H "Authorization: Bearer $VINDONISSA_API_KEY" \
  https://api.helvetii.ai/task_status/abc-123
# → {"task_id":"abc-123","status":"complete","result":{...},...}
```
