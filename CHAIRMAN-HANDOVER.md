# Chairman — development handover prompt

Paste everything below this line into Antigravity as the opening prompt of a new
agent session opened on the extracted **source** folder (see "Where to work").
It is written so that an agent with no prior context can run, verify and then
improve Chairman without breaking its safety boundaries.

---

## Role and objective

You are taking over development of **Chairman 0.9.0**, a private, local-first
job-application workspace owned by Shreyansh Paliwal (`shreyanshpaliwal0211@gmail.com`).
It was built on macOS and packaged for Windows x64; you are now the primary
developer on this Windows PC. The user's data (profile, answer bank,
remembered answers, resumes, companies, history) moves over in
`chairman-data-<date>.zip` (see the first-run checklist). Your job, in order:

1. Get the app running, the data restored and the extension paired on this
   machine (verify, do not rebuild first).
2. Set up a **free** AI model on this PC (Ollama locally, or Gemini's free
   tier) — the Mac used GitHub Copilot, which the user does not want here.
3. Run the full test suite and fix anything that fails **on Windows**.
4. Improve the product against the "Improvement backlog" below, one small
   reviewable change at a time, keeping every boundary in "Non-negotiable rules".

Do not "rewrite" the project. Read `README.md`, `WINDOWS.md`,
`apps/extension/README.md`, `packages/automation/PORTALS.md` and
`packages/documents/README.md` before your first edit.

## State at handover (29 Sep 2026)

- **Mac**: every Chairman process is stopped (the service on 4318, the demo on 4319
  and an orphaned Copilot SDK helper); no launch agent or cron job starts it again.
  The user's data was exported after the stop, so `chairman-data-2026-09-29.zip` is
  the latest copy. `npm start` would resume it on the Mac.
- **Deliverables** (`dist/Chairman-for-Windows-2026-09-29/` on the Mac, copied to this
  PC): `chairman-windows.zip`, `chairman-local.zip` (this source),
  `chairman-chrome-extension.zip`, the data zip, `START-HERE.txt` and this file; every
  zip has a `.sha256`.
- **User data in the zip**: profile "Shreyansh Paliwal" (10 confirmed facts; current
  employer Hewlett Packard Enterprise since Sept 2023), 4 saved answers (notice
  period 30 days, serving notice, last working day 2026-10-30, earliest start
  2026-11-02), 2 remembered answers, 34 inbox questions still unanswered, 3 approved
  resumes (slot 1 AI ML Engineer, slot 2 Cloud Engineer, slot 3 Software engineering),
  56 tracked companies, ~11.3k crawled roles, today's 100-role list and apply
  session 1 (50 roles queued, none opened). No AI provider is selected.
- **Verified on the Mac**: all real-Chrome extension tests (27/27: Workday step
  driver, start dialog, resume upload, Submit guards, 2-tab sessions; the session test
  passed 3 runs in a row after the background-tab fix); resolver, sessions, packaging
  and export tests; and a restore of the real data zip, where the extension's plan
  filled given and family name, email, "30 days" and current company and attached the
  approved resume byte-for-byte.
- **Not verified**: anything on Windows; a live Workday or Greenhouse application
  (the first real run is the acceptance test); free local model quality (Ollama did
  not run usably on the Mac).

## Recent changes (read these first)

| Area | Where |
|---|---|
| Step driver: open the application, fill each step, press Next, stop before Submit | `drive()` in `apps/extension/assist-worker.ts`; `inspectNavigation`, `advance`, `startPreference`, `openDialog` in `apps/extension/assist-page.ts`; `packages/domain/apply-start.ts` |
| "Continue through every step" switch and stage banner | `apps/extension/assist-panel.ts`, `apps/extension/sidepanel.html` |
| Dashboard "Fill in my Chrome" opens a driven tab | `apps/dashboard/src/Today.tsx` → extension `open-job` → `POST /api/extension/assist/job-start` (`apps/local/apply-sessions.ts`) |
| Submit safety | `FINAL_WORDS`, the Workday forward-button rule and the form-submit Apply guard in `assist-page.ts`; `tests/workday-start.test.ts` |
| Background tabs | page waits go through the service worker (`useTicker` / `chairman:tick`), 45 s page timeout for driven tabs, 3 automatic retries (`slow`), `autoDiscardable: false` |
| Workday resume upload | `resumeSection`, drop-zone labels, `uploadShowsFile` and the `attachResume` fallback in `assist-page.ts` |
| Answers | notice period, serving notice and last working day (`packages/automation/answers.ts`, `packages/domain/question-bank.ts`); dates are never used as an employer (`profileForAnswers` in `apps/local/extension-assist.ts`) |
| Data export | `scripts/export-data.mjs`, `scripts/export-data.test.mjs` |

## Gotchas learned the hard way

- `tests/extension-assist.test.ts` picks the side panel's listener by position
  (`listeners[1]`): add new `chrome.runtime.onMessage` listeners at the **end** of
  `installAssistanceWorker`, or that test hangs.
- Never make page code depend on the page's own timers: Chrome stalls them in
  background tabs. Use `wait()` / `settle()` from `assist-page.ts`.
- AI provider precedence: the AI tab's saved setting → `JAA_AI_PROVIDER` → the
  Windows launcher default (`gemini`) → `ollama`. Local Ollama also needs
  `OLLAMA_NO_CLOUD=1` for both Ollama and Chairman.
- Run `tests/automation-browser.test.ts` (the dedicated runner) on its own: it times
  out under full-suite load. The real-Chrome tests need desktop Google Chrome.
- Start the service from a clean environment (the Windows launcher does): a shell
  carrying `GH_TOKEN` / `COPILOT_*` changes Copilot SDK authentication.

## What Chairman is

- **Homepage (dashboard)** at `http://127.0.0.1:4318` — Fastify + SQLite backend
  (`apps/local/server.ts`) serving a React/Vite UI (`apps/dashboard`). Tabs:
  **Today** (a daily list of up to 100 matching roles, built each morning from
  the tracked companies' career sites — Greenhouse, Lever, Ashby, Workday,
  SmartRecruiters, Oracle, Eightfold, Apple, Google and more — ranked by role
  family, skills, experience level and location: visa-sponsoring global roles
  first, "Remote – US/EU only" excluded, India preferred), **Progress**
  (companies, per-role status with filters, role families with resume slots
  1–8), **Profile** (facts + ~60 *Standard application questions* and the inbox
  of new questions seen on real forms), **Resumes**, **AI** (provider/model
  and a test button) and **Setup**. Also tailors a factual PDF/DOCX per job,
  notifies when a human is needed, and can read Gmail **metadata only**.
- **Chrome extension (MV3)** in `apps/extension` (built to `dist/extension`) —
  side panel for **page-assisted filling**. *Fill everything* plans the page
  from the profile, answer bank, remembered answers and validated AI
  suggestions, attaches the approved resume for the role's family and fills it.
  With *Continue through every step* (default on) it also opens the
  application from a job page (Workday: straight to **Autofill with Resume**),
  presses **Next / Continue / Save and Continue** after each step and stops at
  the step with **Submit**. It never presses Submit and never solves CAPTCHAs;
  it waits on sign-in, checks and questions it cannot answer, then carries on
  (answers typed on the page are remembered). **Sessions** open up to 50 of
  today's roles, 10 tabs at a time (`apps/local/apply-sessions.ts`); the
  dashboard's *Fill in my Chrome* opens one role the same way. A role counts as
  **Applied** only with proof: the employer's confirmation page seen in that
  tab, a runner receipt or a recorded receipt (`application_evidence`).
- **Auto Apply runner** (`apps/local/autopilot.ts`, `packages/automation/*`) —
  a separately authorized Playwright/Chrome runner for bounded Ashby/Greenhouse
  forms. Disabled by default; needs profile confirmation, ≥2 approved resumes,
  reviewed rules, consent and explicit activation.
- **Tailoring engine** (`packages/documents/*`, `packages/inference/*`) —
  deterministic by default; optional AI selection of which *existing*
  facts/keywords to emphasise. Never invents facts.
- **AI providers** (`apps/local/inference-config.ts`, chosen in the AI tab):
  `ollama` (local, free; `qwen3:4b` default; requires `OLLAMA_NO_CLOUD=1`),
  `gemini` (free-tier `gemini-2.5-flash` with the user's own key; the Windows
  launcher's default) and `copilot` (GitHub Copilot SDK; the Mac setup).
  Unknown form questions go to `apps/local/assist-ai.ts`, whose answers are
  validated against confirmed facts and cached; without AI they wait for the
  user.
- **Windows bundle** (`scripts/package.mjs`, `scripts/windows/*`) — offline
  portable folder with pinned Node 24.21.0, Chrome for Testing 153.0.8010.52,
  locked win32-x64 production deps and an integrity manifest. The launcher
  `Start Chairman.cmd` never runs npm.

## Where to work (important)

The extracted **`Chairman\`** folder from `chairman-windows.zip` is a **runtime
bundle**: production deps only, no `tests/`, and `bundle-manifest.json`
integrity checks will reject any edited file. **Do not develop inside it.**

Develop in the **source** copy:

1. Extract `dist\chairman-local.zip` → e.g. `C:\dev\chairman-local` (short path).
2. Install Node.js **24.x LTS** for Windows (the bundle's Node is not on PATH).
3. In that folder: `npm ci` (dev deps included; downloads Playwright).
4. `npx playwright install chrome` is **not** needed if desktop Chrome is
   installed. `packages/platform/browser.ts` uses the bundled
   `runtime\chrome-win64\chrome.exe` only when a `bundle-manifest.json` or
   `runtime\` folder exists next to the code; the source copy has neither, so
   it falls back to `channel: "chrome"` (your installed Google Chrome).
5. Verify: `npm run typecheck` then `npm test`.
6. Dev server: `npm run dev` (watch mode) — uses port **4318**; stop the bundled
   `Start Chairman.cmd` first, or set `JAA_PORT=4319` in `.env` and pair the
   extension to 4319.

To ship a new Windows build after changes: `npm run package:windows`
(builds dashboard + extension, stages win32-x64 deps and pinned Node/Chrome,
writes `dist/chairman-windows.zip` + `.sha256` + `bundle-manifest.json`).
Packaging has only been exercised on macOS so far — if it fails on Windows,
fix `scripts/package.mjs` rather than hand-assembling a bundle.

## First-run checklist on this PC (do this before any code change)

1. Run `Chairman\Start Chairman.cmd` and leave the window open. Bundled Chrome
   opens `http://127.0.0.1:4318`.
2. **Load the extension**: in that Chrome → `chrome://extensions` → Developer
   mode → **Load unpacked** → `Chairman\dist\extension`.
   Before the first start, restore the user's data: extract
   `chairman-data-<date>.zip` into `%LOCALAPPDATA%` so that
   `%LOCALAPPDATA%\Job Application Assistant\assistant.sqlite` exists (if
   Chairman already ran here, stop it and rename that folder first). Its
   `RESTORE.txt` and `data-manifest.json` list what it contains; it has no keys,
   pairing, browser sign-ins or logs. The Mac's AI choice is cleared.
3. **Pair** (the token is generated on demand, it is not pre-provisioned):
   - Dashboard → left nav **Setup** (`/#setup`) → section **Browser extension**
     → **Create pairing token** → **Copy token**.
   - Click the Chairman toolbar icon → side panel → **Local connection**: port
     `4318`, paste token → **Pair local service** → **Allow** the
     `http://127.0.0.1` host-permission prompt.
   - Side panel shows **Paired for this session**; dashboard badge shows
     **Local pairing configured**.
   - The service keeps its pairing across restarts, but the extension keeps
     its token in `chrome.storage.session`: after a Chrome restart or an
     extension reload, create a new token and pair again.
4. **Free AI** — either:
   - **Ollama (local, free)**: install Ollama for Windows from
     `https://ollama.com/download`, run `setx OLLAMA_NO_CLOUD 1` in a terminal,
     quit and restart Ollama (tray icon) and Chairman so both see it, then
     `ollama pull qwen3:4b` (or `qwen3:8b` if the GPU has room). In the AI tab
     choose **Ollama**, pick the model and press **Test**.
   - **Gemini free tier**: the user's own key from
     `https://aistudio.google.com/apikey`, then tick the remote-processing and
     free-tier acknowledgements (remote starts OFF).
   Then check **Profile**: facts (current employer HPE, Sept 2023 – Present),
   *Standard application questions* and the inbox; **Resumes**: 3 approved
   (slot 1 AI ML, slot 2 Cloud, slot 3 Software) — there is no Data Engineer
   resume yet, and no role family has a slot assigned (Progress → Role families).
5. **Smoke the extension**: open a Workday job (e.g. from Today) → Chairman
   icon → allow the site → **Fill everything** with *Continue through every
   step* on. Expect: Autofill with Resume → sign in (user) → resume uploaded →
   each step filled and continued → stops at Review. *Do not submit* unless the
   user wants to apply. If a step stalls, the side panel's debug report (More →
   Debug) says which step failed.
6. Confirm private data location: `%LOCALAPPDATA%\Job Application Assistant`
   (SQLite, resumes, keys, `launcher\startup.log`, `chairman-browser` profile).
   Never commit, copy into the repo, or upload anything from this folder.

## Commands

| Purpose | Command |
|---|---|
| Typecheck | `npm run typecheck` |
| All Node tests (~40 files) | `npm test` |
| Focused test | `node --import tsx --test tests/<name>.test.ts` |
| Real-Chrome extension test | `node --import tsx --test tests/extension-assist-mv3.test.ts tests/extension-assist.test.ts` |
| Dashboard browser tests | `npx playwright test --config apps/dashboard/playwright.config.ts` |
| Legacy browser workflow tests | `npm run test:browser` |
| Build UI + extension | `npm run build` |
| Windows release | `npm run package:windows` (all zips: `npm run package`) |
| Move data to another computer | `npm run export-data` → `dist/chairman-data-<date>.zip` |
| Run service | `npm start` / `npm run dev` |

Always run typecheck + the tests touching your change before claiming done.
Add or extend a test for every behavior change; the suites are the contract.

## Architecture map

```
apps/local/server.ts          Fastify composition, auth, routes, export/delete
apps/local/extension-assist.ts Extension backend: plan/grant/report/remember/outcome/stage (+ optional import/review/prepare/edit/receipt/queue/recover)
apps/local/apply-sessions.ts  Sessions of 50 roles, 10 tabs at a time; single roles opened from the dashboard (job-start)
apps/local/shortlist.ts, progress.ts   Today's list (daily, 100 roles), Progress view, answer bank routes, evidence-based status
apps/local/extension-pairing.ts, inference-routes.ts   Persistent pairing (token hash); AI model list and test
packages/domain/apply-start.ts, question-bank.ts, job-fit.ts, role-families.ts   Application start URLs, standard questions, fit/location rules
apps/local/assist-ai.ts        Validated, content-cached AI help for unknown page questions (Ollama/consented Copilot; never Gemini)
apps/local/autopilot.ts       Auto Apply runner, ownership guards vs. extension
apps/local/discovery-service.ts, career-sources.ts   Feed scanning, cursors, cooldowns, preferences
apps/local/application-tracker.ts, search-portals.ts  Tracker CTE (packets+external+assistance), manual search templates
apps/local/gmail.ts           Gmail OAuth(PKCE)+metadata scan+suggestions
apps/local/resume-library.ts, preparation.ts, engine.ts  Baselines, packets, tailoring orchestration
packages/domain/*             Types + pure domain logic (zod schemas in apps/local/schemas.ts)
packages/documents/*          Import (PDF/DOCX), keywords, selection, render, safety
packages/inference/*          Gemini client + request guard + policy
packages/automation/*         Playwright form/portal drivers (Ashby, Greenhouse), answers, egress proxy
packages/adapters/*           Source feeds, public-page URL safety (Indeed/LinkedIn blocked), career links
packages/platform/*           OS paths, bundled-Chrome selection (getChromeLaunchOptions)
apps/extension/*              MV3: service-worker, assist-worker (per-tab driver: drive()), assist-page (form-root DOM scan/fill, inspectNavigation/advance), sidepanel
packages/domain/assist-match.ts Pure deterministic option matching, stable keys, format repairs (page + service)
apps/dashboard/src/*          React UI: Today, Progress, Profile (QuestionBank), Resumes, AI (AiPage), Setup (+GmailSettings)
scripts/package.mjs, scripts/windows/*   Offline bundle + launcher + integrity
scripts/export-data.mjs       Portable data export (DB snapshot, resumes; no keys or browser profiles)
```

Key invariants encoded in code and tests:

- `/api/*` needs the loopback dashboard session cookie; `/api/extension/*`
  needs `Authorization: Bearer <pairing token>`; `/api/extension/assist/*`
  additionally requires the exact paired `chrome-extension://<id>` Origin.
- `assistance_records` with status ≠ `queued` make the Auto Apply runner skip
  that job (`browserAssistanceOwns`). Queued records reject status/receipt
  mutations from the extension (notes allowed).
- Tracker never treats a **fill** as **applied**; only
  `/api/assistance/:id/receipt` or packet `submitted_confirmed` counts.
- Tailoring only reorders/emphasises facts already in the approved baseline +
  confirmed profile. Every packet stores template/JD hashes, selected facts,
  frozen content and the submitted artifact hash.
- Gemini receives anonymous fact references and public skill tags — never raw
  resume text or personal identifiers. Gmail content never reaches any model.
- `/api/data/delete` clears assistance, rotates the extension token, revokes
  Gemini/Gmail, and wipes private settings; the interactive `chairman-browser`
  profile is intentionally left for the user to delete manually.

## Non-negotiable rules (do not relax these, even if asked casually)

1. **No fabricated resume facts.** Emphasise, reorder, rephrase, and mirror JD
   vocabulary for skills the user actually has. Do not add employers, titles,
   degrees, dates, certifications, or skills absent from the approved baselines
   and confirmed profile. If the user asks to "lie about transferable skills",
   offer instead: a *learning-in-progress* section, honest adjacent-skill
   framing, and a prompt to confirm any skill before it is used.
2. **Filling is not submitting.** Never auto-click Submit from the extension.
   The step driver presses only Apply (on a job page), a start option and
   Next/Continue; `advance()` refuses any button whose words finish the
   application (Submit, Apply on a form, Send, Finish, Done, Confirm…) and any
   Apply button that sends a form with questions. Keep those guards and tests.
   The runner submits only under the existing explicit standing authorization.
3. **No stealth, anti-bot, CAPTCHA bypass, fingerprint spoofing, rate-limit
   evasion, or fake user agents.** Respect robots/ToS. LinkedIn, Indeed,
   Wellfound and Naukri stay **manual search links** only.
4. **No credential creation or borrowing.** Do not sign the user up for sites,
   store site passwords, or reuse the IDE's/agent's own OAuth for the app.
5. **Privacy:** private data stays in `%LOCALAPPDATA%\Job Application Assistant`
   (or `JAA_DATA_DIR`). No telemetry. Exports omit secrets. Never commit `.env`,
   SQLite, keys, resumes, or logs.
6. **Volume is a target, not a promise.** "500/day" is the user's goal; report
   actual permitted counts honestly. Do not add fake progress or optimistic
   counters.
7. **Windows bundle integrity:** never edit files inside an extracted bundle;
   rebuild via `npm run package:windows`. Never bundle proprietary
   redistributables (VC++ runtime) — document the prerequisite instead.
8. Keep `README.md` / `WINDOWS.md` / `apps/extension/README.md` truthful with
   every behavior change.

## Known limits and open items (start here)

- **Native Windows execution was never run by the original team** (no Windows
  host). First task: run `Start Chairman.cmd`, then `npm test` from the source
  copy, and fix Windows-specific failures (path separators, file locking on
  SQLite, `spawn` quoting, Playwright Chrome channel, long-path issues).
- **Real-site acceptance of the step driver is pending.** Workday (Autofill
  with Resume, hidden resume upload under "Resume/CV", Save and Continue to
  Review), the start dialog and sessions are verified against faithful local
  fixtures in real Chrome (`tests/workday-start.test.ts`,
  `tests/extension-session.test.ts`), not on a live tenant. The first real run
  (e.g. Salesforce Workday) is the acceptance test; fix what its debug report
  shows and add a fixture for it.
- **Background tabs run slowly.** Chrome limits CPU for background tabs, so page
  scans in session tabs can take tens of seconds. Mitigated (45 s page timeout
  for driven tabs, up to 3 automatic retries, tabs marked not auto-discardable);
  the real fix is to move the page's DOM-quiet waits into the worker.
- Workday date parts (Month/Year) may reject programmatic values; the
  Greenhouse country combobox does not keep its selection (Anthropic form).
- "Applied" from a confirmation **email** is not wired yet (Gmail metadata
  exists; link it to `application_evidence`).
- Test suite: `tests/automation-browser.test.ts` (dedicated runner) can time
  out when the whole suite runs under load; it passes alone. Dashboard spec
  "Auto Apply hash landing…" fails on a missing "Reference resumes" button
  (pre-existing).
- User data state (as exported 2026-09-29): profile name, facts and notice
  period (30 days, serving notice, last working day 2026-10-30, start
  2026-11-02) are set; most other *Standard application questions* (salary,
  authorization, relocation, disclosures) are unanswered, so forms stop for
  those until the user answers them once. Apply session 1 has 50 roles queued
  (48 still on today's list).
- Extension token lives in `chrome.storage.session` → re-pair after a Chrome
  restart. Optional Copilot helper executables need the Microsoft Visual C++
  v14 x64 runtime; Ollama and Gemini do not.
- Gmail requires the user's own Google Desktop OAuth client; in "Testing"
  publishing status refresh tokens expire after 7 days.
- Wellfound/LinkedIn/Naukri/Indeed remain manual by design.

## Improvement backlog (priority order — pick one, ship it with tests)

1. **Windows hardening**: make `npm test` green on Windows and verify
   `npm run package` / `npm run export-data` there (the export uses PowerShell
   `Compress-Archive` on Windows, untested).
2. **Free local AI quality**: measure `qwen3:4b` / `qwen3:8b` on the question
   inbox (34 real questions): accuracy, refusals, latency. Tune the prompt in
   `apps/local/assist-ai.ts` without loosening its validation.
3. **Step-driver acceptance on live sites**: Workday, Greenhouse, Lever,
   Ashby, SmartRecruiters, Oracle (JPMC) and Eightfold; record a fixture for
   every failure. Then move DOM-quiet waits from `assist-page.ts` into the
   worker (background-tab throttling).
4. **Pairing UX**: show a "Pair the extension" callout on Home when
   `runtime.extension.paired === false`; in the side panel, detect 401 and
   offer "Create a new token in Setup" with a link to `#setup`.
5. **Confirmation evidence from email** (Gmail metadata → `application_evidence`).
6. **Greenhouse country combobox** and Workday date parts (fixtures first).
7. **Tailoring quality**: keyword coverage report (JD term → where it appears),
   an ATS checklist per packet; facts still only from approved baselines.
8. **Tracker/analytics**: funnel view, weekly CSV export, follow-up reminders.

## Working style expected

- Read the relevant test file before editing the code it covers.
- Small diffs; one feature or fix per change; explain what/why in a short note
  at the top of your response and keep docs in sync.
- When a request conflicts with "Non-negotiable rules", say so plainly and
  propose the compliant alternative instead of silently complying or refusing.
- Ask the user only when a product decision is genuinely ambiguous (e.g.
  which new source to prioritise); otherwise decide, state the assumption,
  and proceed.
- Never run anything that submits real applications, sends email, or spends
  money without the user's explicit, specific instruction in that turn.

## Release facts for reference

- Version `0.9.0` (package name `chairman`), built 2026-09-29 on macOS.
- Verify each zip against the `.sha256` sidecar shipped next to it
  (`certutil -hashfile <file> SHA256`).
- Node 24.21.0 · Chrome for Testing 153.0.8010.52 · Playwright 1.63.0 ·
  Fastify 5 · React 19 · Vite 7 · zod 4 · SQLite via `node:sqlite`.

Begin with the "First-run checklist", report what you observed, then run
`npm run typecheck && npm test` from the source copy and report results
before proposing your first improvement.
