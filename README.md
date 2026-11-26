# Reel: movie & series picker CRM (FastAPI + MagickVoice)

Users browse top-rated movies and series. When they can't decide, the **Let Riya decide** button at the bottom of every page makes MagickVoice call them. Riya (the `reel_picker_v2` voice agent) asks about their mood, who's watching and how much time they have, then pitches one or two titles. After the call, the picks show up under **Suggestions** in the nav bar, and a MagickVoice automation sends them on **WhatsApp**.

```
User ──"Let Riya decide"──► POST /api/calls ──► MagickVoice POST /proxy/calls  (prompt template + caller id + automation_ids)
                                                     │  Riya calls the user (~2 min)
                                                     ▼
                         Automation "After analysis is ready"
                           ├─ Step 1  WhatsApp → user's phone: the picks
                           └─ Step 2  Webhook  → POST /api/webhooks/magickvoice?token=…
                                                     │
CRM ◄────────────────────────────────────────────────┘  saves the call, outcome, summary and suggestions
    (on localhost without a public URL, the CRM polls GET /proxy/calls/:id instead)
```

## Quick start

```bash
cd reel_crm
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env      # fill in MAGICKVOICE_API_KEY and MAGICKVOICE_PROMPT_ID (see below)
uvicorn app.main:app --reload   # http://localhost:8000
```

Sign up at `/signup` (name, email, phone, password). API docs are at `/docs`. Data lives in `data/reel.db` (SQLite). The 86-title catalog in `data/catalog.json` loads on startup and is the same list Riya recommends from.

Try it without MagickVoice: sign up, then run

```bash
python scripts/simulate_webhook.py --email you@example.com --titles "Coco | Paddington 2" --chosen Coco
```

Tests: `pytest -q`.

## Pages

| Page | What it shows |
|---|---|
| **Browse** (`/`) | Featured title, search, Movies/Series filter, mood chips, rows by audience (Kids & family, Gen Alpha, Telugu & Hindi, Anime & K-drama, Gen Z). Clicking a title opens details; **+** saves it to My list |
| **Suggestions** (`/suggestions`) | One card per call: date, mood, who's watching, outcome, WhatsApp status, Riya's summary and the titles she pitched, with **You chose this** and **Mark watched** |
| **Profile** (`/profile`) | Name, phone Riya calls, age group, languages, genres, loved and disliked titles, WhatsApp opt-in, My list and call history |
| **Let Riya decide** | Fixed bar at the bottom. Confirm the number and give consent, then a live status (Ringing, On call, Picking, Ready) and the picks when the call ends |

## MagickVoice setup

### 1. `.env`
| Variable | What to put |
|---|---|
| `MAGICKVOICE_API_KEY` | MagickVoice → Account → API keys (`mgkp_…`) |
| `MAGICKVOICE_TENANT_ID`, `MAGICKVOICE_ACCOUNT_ID` | Already set for the staging workspace |
| `MAGICKVOICE_PROMPT_ID` | The id of the Reel agent made from `reel_picker_v2.json` |
| `MAGICKVOICE_CALLER_ID` | The number users see |
| `MAGICKVOICE_AI_PIPELINE` + voice fields | `platinum` or `custom`, same as the screening CRM |
| `MAGICKVOICE_WHATSAPP_AUTOMATION_ID` | The automation from step 2 |
| `PUBLIC_BASE_URL` | A public `https://` URL (ngrok or cloudflared) so webhooks reach you. On localhost the CRM polls instead |

Every call sends `prompt_variables` (and the same values in `recipient.metadata`): `caller_name`, `age_group`, `preferred_languages`, `favourite_genres`, `loved_titles`, `disliked_titles`. Add `{{caller_name}}` and the others to the prompt if you want Riya to use them. Unused variables are ignored.

### 2. WhatsApp automation (one time)
MagickVoice → **Automations → New Automation**:
1. **Trigger:** *After analysis is ready*.
2. **Step 1: WhatsApp** to `{{callee.phone}}`. Use an approved template, for example:
   *"Hi {{callee.name}}! Riya's picks for tonight: {{analysis.dimensions.suggested_titles}}. You chose: {{analysis.dimensions.chosen_title}}. See them all: {{callee.metadata.suggestions_url}}"*
   Optionally add an *Only run if* `analysis.dimensions.call_outcome` is not `wrong_number` / `not_interested` / `no_response`.
3. **Step 2: Webhook.** URL = `PUBLIC_BASE_URL/api/webhooks/magickvoice?token=WEBHOOK_TOKEN`, method POST, default body.
4. Save, and put the automation id in `MAGICKVOICE_WHATSAPP_AUTOMATION_ID`.

The CRM attaches this automation to every call unless the user turned WhatsApp off in Profile. Because the Webhook step runs after the WhatsApp step, a webhook that carries the analysis marks WhatsApp as **Sent**. To report it yourself, add `"whatsapp_status": "sent"` (or `failed`) to a custom webhook body.

> Use the automation's **Dry-run → A recent run** to check the exact template variable names in your workspace. The CRM reads any casing of the dimension keys.

### What the webhook does
* Matches the call by MagickVoice call id, then `external_ref_id`, then `callee.metadata.user_id`.
* Reads `call_outcome`, `suggested_titles` (split on `|`), `chosen_title`, `mood`, `watch_group` and `call_summary` (the dimensions in `reel_picker_v2.json`).
* Matches each title to the catalog loosely ("Doraemon 2" finds *Stand By Me Doraemon 2*). Titles that don't match are still saved by name.
* Retries are deduplicated with `Idempotency-Key`. Auth is `?token=` or a valid `X-Magick-Signature`.

## Layout

```
app/main.py            FastAPI app, sessions, page routes
app/config.py          .env settings
app/db.py              SQLAlchemy models: User, Title, WatchlistItem, Call, Suggestion, WebhookEvent (+ catalog seed)
app/security.py        password hashing, phone normalising, current_user
app/magickvoice.py     API client, call payload, webhook auth, payload parsing
app/services/calls.py  start a call, apply webhooks / polling, save suggestions
app/schemas.py         request bodies + JSON output
app/routers/           auth.py, titles.py, profile.py, calls.py (calls, suggestions, webhook)
app/static/pages/      browse, suggestions, profile, login, signup (.html)
app/static/js/         common.js (nav, posters, call flow), one file per page
app/static/css/        style.css
data/catalog.json      86 titles (same as Riya's catalog)
scripts/simulate_webhook.py, tests/test_app.py
```

Guardrails: one active call per user, a cooldown between calls (`CALL_COOLDOWN_SECONDS`), consent checkbox before calling, and users only ever see their own calls.
