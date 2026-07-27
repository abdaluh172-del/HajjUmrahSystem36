# Hajj & Umrah Sentiment Analysis — Backend (Flask + real ML model)

This is a **real** working backend: a scikit-learn model (TF-IDF + Multinomial
Naive Bayes, trained on `dataset.py`) served through a Flask REST API, backed
by a SQLite database (`hajj_umrah.db`, created automatically on first run).

It matches the design in the graduation project document (Chapter 1.6
Methodology: TF‑IDF feature extraction + Naive Bayes classification).

## 1. Open in VS Code
Open this `backend` folder in VS Code (`File → Open Folder…`).
Make sure the **Python extension** is installed.

## 2. Create a virtual environment
Open a terminal in VS Code (`` Ctrl+` ``) and run:

```bash
python -m venv venv
```

Activate it:
- Windows (PowerShell): `venv\Scripts\Activate.ps1`
- Windows (cmd): `venv\Scripts\activate.bat`
- macOS / Linux: `source venv/bin/activate`

VS Code may prompt "Select Interpreter" — choose the one inside `venv`.

## 3. Install dependencies
```bash
pip install -r requirements.txt
```

## 4. (Optional) Retrain the model
A model is trained automatically the first time you run `app.py` if
`model.pkl` doesn't exist yet. To retrain manually (e.g. after editing
`dataset.py`) and see accuracy/precision/recall:

```bash
python train_model.py
```

> The included dataset is intentionally small (~90 labeled examples) for
> demonstration. Accuracy will be limited — for a stronger model, replace
> `TRAIN_DATA` in `dataset.py` with a larger, real labeled dataset (hundreds/
> thousands of comments), which is exactly what your project proposes to
> collect in Graduation Project 2.

## 5. Run the API — and the full website
```bash
python app.py
```
The API starts at **http://localhost:5000** and auto-creates/seeds
`hajj_umrah.db` on first run.

**Open http://localhost:5000 in your browser** — this now serves a complete,
real, working website (login → dashboard → analyze comment → comments →
settings), built with React (via CDN, no npm/build step needed) and wired
directly to this Flask API and the trained ML model. Every comment you
analyze is saved for real in `hajj_umrah.db`.

**Default login** (auto-seeded on first run): `abdullah2222@ghjj.sa` / `A1231234` — display name: **Abdullah Alharbi**. Logged-in sessions are remembered in the browser (localStorage) for 12 hours.

## What's new in v9
- **Login Logs (admin-only):** every sign-in attempt (and every signup) is stored
  permanently in the database — name, email, date, time and status
  (success/failed). A new **Login Logs** page in the admin sidebar shows them
  with search, sorting (newest/oldest/name/email/status) and pagination.
  API: `GET /api/login-logs` (admin token required).
- **Comments Language (admin-only, in Settings):** choose Original / Arabic /
  English — all displayed comments are translated automatically in the browser
  to the chosen language (originals in the database are never modified).
- **Admin-only tools:** Export CSV (Comments), Print / Save PDF (Reports) and
  the whole **Analytics** page are now visible to the admin only. A non-admin
  reaching Analytics, Login Logs or Users sees an **Access Denied** screen.
  `GET /api/comments/export` now requires the admin token on the server too.
- **Comments language (v11 — per account):** in Settings, EVERY signed-in
  account (regular users AND the admin) can choose the comments display
  language — Original / Arabic / English. Displayed comments are translated
  automatically in the browser; originals in the database are never modified.
  The choice is personal — saved on the account's own row via
  `PUT /api/me/comment-lang`, so it never affects other users, is restored
  on every login (any device) and survives restarts/redeploys. Guests don't
  have an account, so they always see originals.

### v15.4 — Real Google Sign-In + Arabic sentiment accuracy fixes
- **Google Sign-In is real** (`app.py`): `POST /api/auth/google` now verifies
  the credential Google's JS library hands back (audience = this app's
  client id, issuer = Google, email verified), then finds-or-creates the
  account by that verified email and signs in through the exact same
  `make_token()`/session system as normal email/password login — same
  30-day "remember me", same everything downstream. Set
  `GOOGLE_OAUTH_CLIENT_ID` on Render to turn it on (see render.yaml for the
  Google Cloud Console steps); without it the login page's Google button
  stays exactly as it was ("coming soon"). No client secret needed — this
  uses Google's JS "Sign In With Google" button (an ID token handed back in
  the browser), not a redirect-based OAuth code exchange, so there's
  nothing else to configure beyond the client id and its authorized
  JavaScript origins. Apple sign-in is untouched, still a documented stub.
- **Session persistence was already correct** — `saveSession`/`loadSession`
  (localStorage for "remember me", sessionStorage otherwise) plus the
  backend's signed, expiring token already did exactly what "stay signed in
  across visits" needs. Google sign-in reuses that same system as-is.
- **Fixed the real cause of "ممتاز" (excellent) scoring as neutral**
  (`sentiment.py`): traced to a spelling-variant gap, not a logic bug — the
  lexicon had "سيء" (bad) but not "سيئ" (a different, very common hamza
  placement for the same word), so that specific spelling scored 0 and the
  comment fell through to neutral. Fixed with a normalization step applied
  to the whole lexicon at once (hamza/alef/ya variants folded to one form
  before every lookup) rather than hand-adding one spelling — confirmed
  zero collisions across the existing word list, and confirmed the same
  gap silently affected other existing entries too ("افضل" for "أفضل",
  "اسوأ" for "أسوأ").
- **Fixed intensifiers placed after the word they modify** (`sentiment.py`):
  "ممتاز جدا" scored identically to plain "ممتاز" — the code only checked
  for "very" BEFORE the adjective (English order), never after (the common
  Arabic order). Now checks both directions.
- **Hardened the shared LLM prompt** (`ai_pipeline.py`, used by
  Anthropic/OpenAI/Gemini alike): added an explicit rule that a short
  comment, even a single word, gets scored with the same confidence as a
  full sentence — brevity is never a reason to default to "neutral".
- All of the above were verified against the reported example and the
  other short phrases it was reported alongside (ممتاز، رائع، سيئ، سيئة،
  ممتاز جدا، أنصح به، لا أنصح به) — every one now classifies correctly.
  Moderation wordlists, the relevance heuristic, and the trained ML model
  were deliberately left untouched this round, per the request to change
  nothing outside these three items.

### v15.3 — Responsive design pass (mobile / tablet / laptop)
- The app shell (collapsible off-canvas sidebar on mobile with a tappable
  backdrop, static always-visible sidebar from `md:` up, RTL/LTR-aware, the
  login page's two-panel split, filter bars, and comment cards) was already
  solidly responsive on inspection — this pass audited the whole file
  page-by-page against real breakpoint behavior rather than assuming, and
  fixed the specific gaps that turned up:
  - **Users table couldn't scroll horizontally** (`UsersPage`): the other
    two data tables in the app were already wrapped in `overflow-x-auto`;
    this one wasn't, so on a narrow phone its 4 columns (Name/Email/Role/
    Actions) had nowhere to go but get clipped by the card's
    `overflow-hidden`. Wrapped it the same way as the other two, and added
    `whitespace-nowrap` to the header row to match.
  - **Two grids forced 3 columns at every width**, including a 320px phone:
    the "How it works" 3-step explainer (icon + title + description per
    cell — genuinely cramped at that width) and the new AI-providers badge
    row added in v15.2 (which sat right next to the External Sources panel
    but didn't share its `sm:`-and-up breakpoint, so the two looked
    inconsistent on the same page). Both now stack to one column below the
    `sm:` breakpoint like their neighbors already did.
  - **Two loading-state placeholders used flat `p-6`** (Dashboard,
    Analytics) instead of the `p-4 md:p-6` every other page and every other
    state on those same two pages already used — a small but real
    inconsistency between the "loading" and "loaded" view of the same page.
  - **Country/city breakdown rows had no truncation**: a long place name
    could push the count off the row on a narrow card; added `truncate` +
    `min-w-0` on the name and `shrink-0` on the count, the same shrink/
    truncate pairing already used for comment author names and the
    assistant page's header.
  - **Added `overflow-x:hidden` on `html`/`body`** as a defense-in-depth
    backstop — no known cause left to trigger it after the fixes above, but
    it costs nothing and guards against any future one-off overflow (a
    shadow, a rounding error, a browser quirk) turning into a page-wide
    horizontal scrollbar.
  - Everything else audited and left alone deliberately: the sidebar/topbar
    shell, all filter bars, comment cards, the login split-panel, Settings/
    Profile (already `max-w-md`), and the Dashboard's 2-then-4-column stat
    grid were already correct — changed only what had a concrete, traceable
    reason to change, not a general-purpose rewrite.
- No backend changes, no new dependency — pure `static/index.html` layout
  fixes, verified structurally (balanced tags/braces against the file's own
  pre-edit baseline) since this sandbox can't render a live browser preview.

### v15.2 — Google Gemini as a third LLM provider
- **Gemini joins the TIER A LLM pool** (`ai_pipeline.py`, `assistant.py`): set
  `GEMINI_API_KEY` (or `GOOGLE_API_KEY`) on Render and both the comment
  analysis pipeline and the "تعليمات الحج والعمرة" assistant use it —
  exactly the same "bring your own key, no lock-in" pattern as
  `ANTHROPIC_API_KEY`/`OPENAI_API_KEY`, no new dependency (talks to the
  Gemini REST API directly with the existing `requests` package, same as
  the other two providers). If more than one key is set at once, priority
  is Anthropic -> OpenAI -> Gemini.
- **Model default:** `gemini-flash-latest` — Google's own always-current
  alias for its recommended fast/cheap model, used deliberately instead of
  a dated pin so this tier can't repeat the exact "silently falls back to
  Tier B forever" bug the v15.1 fix addressed for the other two providers,
  the moment Google eventually retires today's model. Still overridable via
  the shared `LLM_MODEL` env var.
- **Thinking switched off:** Gemini's 2.5+/3.x models "think" by default,
  and — unlike the fast Claude/GPT models used here — those thinking tokens
  are billed against the *same* `maxOutputTokens` budget as the visible
  answer. Left on, a small budget (this app uses 300 tokens for
  classification and 1000 for assistant replies, matching the other two
  providers) can be consumed entirely by invisible thinking, coming back as
  an empty response that looks just like any other failed LLM call.
  `generationConfig.thinkingConfig.thinkingBudget` is set to `0` on every
  Gemini request to prevent this.
- **Moderation-aware safety settings:** the four adjustable Gemini safety
  categories (harassment, hate speech, sexual content, dangerous content)
  are explicitly set to `BLOCK_NONE` on every request — the whole point of
  the moderation tier is to let the model *see* profanity/hate/violence and
  classify it, and the assistant may legitimately need to discuss sensitive
  topics (e.g. crowd-crush safety, medical emergencies) without Google's
  default filter silently emptying the response.
- No new endpoints, no new dependency, no schema/DB changes — this is
  purely a third option inside the existing LLM tier abstraction.
- **AI status shown like the other integrations:** the Analytics admin page
  used to say "no AI provider configured" as one line of plain text, the
  only integration status on the whole page that didn't look like the
  External Sources panel above it. It now shows the same badge-style
  configured/not-configured cards as YouTube/Reddit/X/Google Maps — one
  card per provider (Claude/GPT/Gemini), with a highlighted ring on
  whichever one is actually answering right now (only one can be active at
  a time under the Anthropic -> OpenAI -> Gemini priority order — a second
  configured key is a live backup, not a second engine running in
  parallel). Backed by the new `llm_providers_configured` field on
  `GET /api/admin/analysis-status` and `GET /api/assistant/status`.
- **Comment analysis made more accurate with NO AI key at all**
  (`sentiment.py`, `ai_pipeline.py`): two fixes to the always-on, no-key
  fallback path, since most of the traffic to this system will run on it
  most of the time:
  - **Fixed a real silent-failure bug in the sentiment fallback:**
    `translation.py` degrades softly on any network error by returning the
    original text unchanged — correct for that file in isolation — but
    tier 2 (VADER) was scoring that "translation" as if it were genuine
    English. VADER's lexicon is Latin-script only, so untranslated Arabic
    silently produced a compound score of ~0 on EVERY comment — a strongly
    negative or positive review would come back mislabeled "neutral" with
    nothing in the logs to explain why, since the call looked like a normal
    success. `sentiment._mostly_latin()` now detects this and drops
    straight to tier 3 (which has a native Arabic lexicon) instead of
    trusting VADER on text it was never able to read. Verified against a
    reproduction of the exact failure (see the project's test notes) rather
    than assumed.
  - **Roughly doubled the category-classification keyword lists**
    (`_CATEGORY_KEYWORDS`): more English synonyms and more Arabic phrasing
    — including common Gulf/Saudi wording pilgrims actually use, not just
    MSA — across all 11 categories, to cut down on comments defaulting to
    "general" just because the heuristic's vocabulary was thin. Ihram/
    Talbiyah-type terms were deliberately kept OUT of the Hajj-specific
    bucket since those rituals apply to Umrah too and would have
    mis-tagged Umrah comments as Hajj ones.
  - Moderation wordlists, the relevance heuristic and the trained ML model
    were left untouched this round — they were already reasonably solid on
    inspection, and the two fixes above are the changes with real evidence
    behind them. Worth a dedicated look later if specific mislabeled
    comments turn up.

### v15 — AI assistant, Google Maps + smarter Reddit data, classification, dedup, full language support
- **"تعليمات الحج والعمرة" AI assistant** (`assistant.py`, `knowledge_base.py`): a ChatGPT-style
  page locked to Hajj/Umrah/Haramain-service topics only — politely declines anything else.
  Reuses `ANTHROPIC_API_KEY`/`OPENAI_API_KEY` (no new keys needed); without one it still
  answers on-topic questions from a small curated knowledge base instead of going blank.
  Structured answers (heading, steps, warnings, du'as, related-info close), conversation kept
  client-side for the session (nothing persisted server-side). New endpoints:
  `POST /api/assistant/chat`, `GET /api/assistant/status`.
- **Google Maps Reviews** (`google_maps_source.py`): schema-ready for the fields in the spec
  (place name/type, country, city, place URL, star rating, review date, language, text,
  username, review id). Automatic fetch works once `GOOGLE_MAPS_API_KEY` +
  `GOOGLE_MAPS_PLACE_IDS` are set (official Places API, no scraping); a manual/admin import
  path (`POST /api/admin/import-reviews`) works today with no key, for CSV/export-based
  imports. Every review is relevance-filtered before it's stored.
- **Smarter Reddit fields** (`external_sources.py`): thread title, subreddit, votes, comment
  count and permalink are now captured (stored in `comments.source_meta`), shown on Comments
  cards.
- **Smart classification** (`ai_pipeline.py`): every comment (site + all external sources) is
  now classified into the platform's topic taxonomy — customer service, service quality,
  transportation, accommodation, cleanliness, crowd management, accessibility, Grand Mosque
  experience, Prophet's Mosque experience, Hajj experience, Umrah experience — via the LLM
  tier when configured, or a keyword-scored heuristic otherwise. Old category values on
  existing rows are left completely untouched; the Analyze/Comments composer defaults to
  "Auto (AI decides)" but can still be set manually.
- **Mixed sentiment**: `positive` / `negative` / `neutral` / **`mixed`** (a comment with
  substantial positive AND negative opinion at once) everywhere sentiment is shown, filtered,
  or aggregated.
- **Cross-source de-duplication** (`dedup.py`): the same opinion posted on more than one
  platform (e.g. Google Maps and Reddit) is recognized via a content fingerprint + fuzzy match
  and skipped instead of double-counted. This is text-similarity de-duplication (near-identical
  wording), not full semantic/paraphrase matching — an honest limitation given no embeddings
  model is used.
- **Extended analytics** (`GET /api/analytics/overview`, admin-only): category breakdown,
  source comparison, country/city comparison (from Google Maps data), a 30-day sentiment
  trend, and a lightweight extracted-suggestions list — alongside the existing dashboard stats.
- **Full language support**: Arabic and English stay fully hand-translated; Turkish, Urdu,
  Hindi and Hebrew now have the highest-visibility UI strings (navigation, auth, buttons, the
  assistant) hand-translated, falling back to English for the long tail of less-common strings
  — an explicit, honest trade-off rather than a large risk of mistranslated text. Comments and
  every AI assistant answer are fully translated regardless (server-side + Google Translate).
  Site UI language, comments display language, and the AI assistant's answer language are each
  an **independent** per-account setting (`PUT /api/me/language-prefs`,
  `PUT /api/me/comment-lang`), matching the spec.
- **"Remember me"**: login/signup accept `remember` (default on); checked keeps you signed in
  up to 30 days (`localStorage`), unchecked keeps the session only for the current tab/browser
  session (`sessionStorage`, ~12h token). The choice is embedded in the signed token itself, not
  just enforced client-side.
- **Google / Apple sign-in — prepared, not yet connected**: `POST /api/auth/google` and
  `POST /api/auth/apple` exist and respond clearly with "not configured" (no fake auth); the
  login page shows both buttons as "coming soon". Wiring up real OAuth later is a matter of
  verifying the client's id_token with real `GOOGLE_OAUTH_CLIENT_ID`/`APPLE_OAUTH_CLIENT_ID`
  credentials — see `render.yaml` for details.
- Every change above is additive: new nullable DB columns via the same
  `ALTER TABLE ... IF NOT EXISTS` pattern already used since v11/v12/v13, so existing databases,
  accounts, comments and settings upgrade automatically with nothing lost.

### v14 — Unified AI pipeline: moderation + relevance (same stack: Flask/Render/your own APIs)
- Every comment (site reviews AND live YouTube/X/Reddit comments) runs the
  full pipeline in `ai_pipeline.py`: detect language -> translate -> AI
  sentiment -> AI content moderation -> Hajj/Umrah relevance -> save ->
  display approved only.
- **Top-accuracy tier (optional, your own key — plain env var, no lock-in):**
  set `ANTHROPIC_API_KEY` or `OPENAI_API_KEY` and ONE cheap LLM call per
  comment handles language, sentiment (real context/sarcasm/mixed-opinion
  understanding), all moderation categories (profanity, insults, hate
  speech, harassment, racism, sexual content, violence, spam) and relevance.
- **Fallback tier (no LLM key):** v13 sentiment engine + HF toxicity
  transformer (with `HF_API_TOKEN`) + always-on local wordlists/spam
  heuristics (extend via `EXTRA_BANNED_WORDS`); relevance heuristics apply
  to EXTERNAL comments only — site reviews are never auto-rejected by the
  cheap heuristic, only the LLM may judge them.
- Flagged (violation) and rejected (off-topic) comments are hidden from all
  non-admin views; irrelevant EXTERNAL comments are dropped before entering
  the database. The admin sees everything via the new Moderation filter,
  with per-flag badges + reason, and can override any AI decision
  (`PUT /api/comments/<id>/moderation`). Re-analyze applies the pipeline to
  old comments; `GET /api/admin/analysis-status` shows provider + counts.

### v13 — Multilingual AI sentiment + server-side translation
- **Tiered sentiment engine** (`sentiment.py`): with `HF_API_TOKEN` set, a
  multilingual transformer (cardiffnlp/twitter-xlm-roberta-base-sentiment)
  classifies every comment natively in its own language via the HF Inference
  API. Without it, the comment is machine-translated to English and scored by
  VADER (negation/intensifiers/emojis aware); a built-in enhanced analyzer +
  the trained ML model cover the final fallback. No keyword-only decisions.
- **Server-side translation** (`translation.py`): every comment — site
  reviews AND external ones from YouTube/Reddit/X/any future source — gets
  its language auto-detected; non-Arabic text is translated to Arabic and
  stored in `text_ar` next to the untouched original (`original_lang` keeps
  the detected code). Works with all languages (en/bn/ur/tr/id/...).
- **Instant display**: the UI shows stored Arabic translations immediately
  (no per-view API calls), defaults to Arabic display, adds a
  "show original / show translation" toggle and a "translated from XX" chip.
  The per-user language choice from v11 still overrides the default.
- **Fixing old data**: admin → Analytics → "Re-analyze comments" re-runs the
  new pipeline over comments stored before v13 (fills translations, corrects
  wrong labels), batched via `POST /api/admin/reanalyze`.
- New admin endpoint `GET /api/admin/analysis-status` shows the active
  engine tier and translation coverage.

### v12 — Real reviews system + external sources (professional upgrade)
- **No more fake comments:** the old seeded sample comments are removed
  automatically on first start (existing real user reviews are untouched),
  and nothing is ever seeded again — the comments table holds only real
  user reviews and comments fetched from official APIs.
- **Real reviews:** signed-in users add their experience with an optional
  **1–5 star rating**; every review stores the author's name and full
  date/time, is analyzed by the AI model instantly, saved permanently in
  the database and listed **newest first**. The reviews page shows the
  **overall average rating**.
- **Likes:** one like per account per review (toggle). Guests can't like.
- **Admin moderation:** publish / hide / edit / delete any comment
  (`PUT /api/comments/<id>/status`, `PUT/DELETE /api/comments/<id>`).
  Hidden comments stay in the database but disappear from non-admin views.
- **AI summary:** `GET /api/insights` aggregates keywords into
  "most mentioned problems" (from negative comments) and "most mentioned
  strengths" (from positive ones) — shown on the Dashboard.
- **Filters & search:** sentiment, stars, source, date range and free-text
  search (also matches author names).
- **External comments (official APIs only — no scraping):**
  `external_sources.py` fetches public opinions about Hajj & Umrah via the
  **YouTube Data API**, **Reddit API** and **X API v2** when their keys are
  set (see `render.yaml` for the env var names). Comments are de-duplicated
  by the platform's own item id, labeled with their source in the UI, and
  auto-refreshed every `FETCH_INTERVAL_MINUTES` (default 60). The admin can
  also fetch on demand from the Analytics page. Without keys the site works
  normally with user reviews only.

### Permanent data storage (v10 — no Persistent Disk needed)
- **All data — user accounts, the admin account, comments and login logs — is
  stored in an external PostgreSQL database** referenced by the `DATABASE_URL`
  environment variable. Nothing is kept in code variables or app memory.
- Because the database lives **outside the app's filesystem**, closing the
  app, a **Restart** or a **Redeploy** on Render's **free plan** never
  deletes anything. Accounts and comments are removed only when the admin
  deletes them manually from the dashboard.
- **Setup (one time):**
  1. Create a free PostgreSQL database on **Neon** (https://neon.tech) —
     free and permanent. *Do not use Render's own free PostgreSQL: it is
     automatically **deleted after 30 days**.* (Supabase is another option.)
  2. Copy the connection string, e.g.
     `postgresql://user:pass@host/dbname?sslmode=require`
  3. In Render → your service → **Environment** → add
     `DATABASE_URL` = that connection string → Save (the service redeploys).
- On first start with an empty database, the app creates the tables, seeds
  the sample comments and the fixed admin automatically. After that, startup
  **never touches existing data** (a changed admin password/name survives
  every restart and redeploy).
- Without `DATABASE_URL` the app falls back to a local SQLite file — for
  local development only (on Render's free plan that file is wiped on
  every redeploy).
You can also use "Sign up" on the login page to create a new account —
accounts are real rows in the `users` table, hashed with werkzeug's
`generate_password_hash` (not plaintext). "Forgot password" is simulated —
it confirms the flow but does not send a real email (no SMTP server is
configured; wire up Flask-Mail or similar for that).

## 6. Test it
```bash
curl http://localhost:5000/api/health

curl -X POST http://localhost:5000/api/analyze \
  -H "Content-Type: application/json" \
  -d "{\"text\": \"الازدحام كان شديد جداً وتأخير في كل شيء\"}"

curl http://localhost:5000/api/comments?per_page=5
curl http://localhost:5000/api/dashboard/stats
```
Or open `http://localhost:5000/api/health` directly in a browser, or use
Postman / VS Code's REST Client extension.

## 7. Connect the React frontend
In the React app (`HajjUmrahSystem.jsx`), replace the client-side
`analyzeText()` calls and the in-memory `comments` state with real `fetch`
calls to this API, e.g.:

```js
const res = await fetch("http://localhost:5000/api/analyze", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ text }),
});
const result = await res.json();
```

Do the same for `/api/comments` (GET/POST/PUT/DELETE) and
`/api/dashboard/stats`. This turns the current in-browser demo into a
frontend properly talking to a real backend + real ML model + real
database, as required by the project scope.

## Endpoints reference
| Method | Endpoint                 | Purpose                                |
|--------|---------------------------|-----------------------------------------|
| GET    | `/api/health`             | Check server + model status            |
| POST   | `/api/analyze`            | Analyze a comment (no DB write)        |
| GET    | `/api/comments`           | List/search/filter/sort/paginate       |
| POST   | `/api/comments`           | Add + analyze + store a new comment    |
| PUT    | `/api/comments/<id>`      | Edit a comment (re-analyzed)           |
| DELETE | `/api/comments/<id>`      | Delete a comment                       |
| GET    | `/api/comments/export`    | CSV export (`?format=csv`)             |
| GET    | `/api/dashboard/stats`    | Aggregated stats for charts            |
| POST   | `/api/assistant/chat`     | v15: Hajj/Umrah AI assistant           |
| GET    | `/api/assistant/status`   | v15: assistant engine status           |
| GET    | `/api/analytics/overview` | v15: categories/sources/geo/trend (admin) |
| POST   | `/api/admin/import-reviews` | v15: bulk-import Google Maps/structured reviews (admin) |
| PUT    | `/api/me/language-prefs`  | v15: save site UI / assistant language |

## Not included (needs further work for production)
- **Authentication (JWT) / roles** — currently the API is open; add
  `flask-jwt-extended` and per-route `@jwt_required()` checks.
- **Larger training dataset** — the model is a real, working classifier, but
  its accuracy is limited by the small illustrative dataset provided.
- **Deployment** — this runs Flask's development server; use `gunicorn` +
  a reverse proxy (nginx) for production.

## Open it from your phone (same Wi-Fi network)
The server listens on `0.0.0.0`, so other devices on the **same Wi-Fi** can
reach it too:

1. On the PC, find its local IP: open PowerShell and run `ipconfig`, look
   for **IPv4 Address** (e.g. `192.168.1.23`).
2. Make sure Windows Firewall allows inbound connections on port 5000 for
   Python (Windows may prompt you the first time you run the server — allow
   it for Private networks).
3. On the phone (connected to the same Wi-Fi), open:
   `http://192.168.1.23:5000` (use your PC's actual IP, not this example).

**Security note:** `debug=True` + `host="0.0.0.0"` exposes Werkzeug's
interactive debugger to everyone on your network — fine for a local
demo/graduation project on a trusted home/campus Wi-Fi, but set
`debug=False` (or don't bind `0.0.0.0`) before using this on any network
you don't fully trust.


## Roles & Permissions (Admin / User / Guest)

The system has exactly three roles:

| Capability | Admin (fixed email only) | Registered User | Guest (no account) |
|---|---|---|---|
| Dashboard, Analytics, Reports | Yes | Yes | Yes |
| View comments | Yes | Yes | Yes |
| Run the analyzer | Yes | Yes | Yes (result not saved) |
| Add comments | Yes | Yes | **No — must sign up** |
| Delete / edit comments | **Yes — admin only** | No | No |
| Users page (see registered emails) | **Yes — admin only** | Hidden + blocked | Hidden + blocked |

- **Fixed admin account:** `abdullah2222@ghjj.sa` / `A1231234` — recreated automatically on startup,
  cannot be deleted, demoted, or have its email changed (enforced server-side).
  These credentials are **no longer shown on the login page** — they live only here and in `app.py`.
- **The admin role is exclusive to that one email.** It can never be assigned through signup or the
  users page (`role` accepts only `user`/`guest`), and on every startup any other row that somehow
  has `role='admin'` is demoted to `user`. Admin endpoints double-check both the role **and** the email.
- **Sign up** always creates a regular `user` account (can view and add comments).
- **Continue as Guest:** the login page has a guest button (`POST /api/auth/guest`) — no account, view only.
  `POST /api/comments` requires a valid `user`/`admin` token, so guests get `401` with an Arabic
  "create an account" message until they register.
- **Admin protection:** login returns a signed token (12h expiry). All `/api/users` endpoints and
  `PUT/DELETE /api/comments/<id>` require `Authorization: Bearer <token>` from the fixed admin;
  everything else gets `403`.
- **Migration:** old databases are upgraded automatically on startup — previously registered accounts
  stored as `guest` become `user`, and any stray admin rows are demoted.
- Set a real `SECRET_KEY` environment variable in production.
