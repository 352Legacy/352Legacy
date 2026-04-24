# Chalk
### An AI copilot for youth sports coaches

*A self-hostable, gift-economy platform for volunteer-run leagues*

> **Working name.** This document uses **Chalk** as a placeholder for the product name. The platform was originally built by and for the 352 Legends youth football program in Citrus County, Florida, but "352 Legends" is an organization the author does not have naming rights to. Final product name is being chosen; the architecture and vision below are independent of the name.

---

## Abstract

Chalk is a full-stack, multi-role, multi-team coaching platform built by a coach, veteran, and dad, for the volunteer coaches of youth sports. It consolidates roster management, skill development, daily training logs, video-verified combine measurement, on-device pose analysis, game-day operations, scouting, and a tamper-proof career record for every kid — into one install a volunteer admin can run on one laptop or one $5/month cloud server.

The platform is open source under AGPL-3.0 and designed to be given away, not sold. It is not a SaaS. It is a **kit**.

This document describes what is built today, the architecture underneath it, the design principles that got it here, and the long-term vision — including the layers that are intentionally deferred until real users reveal which features matter.

---

## Executive Summary

**Problem:** Youth coaches are overwhelmingly volunteers. The tools built for college and pro programs — Hudl, TeamSnap, SportsEngine — scale down poorly in price, complexity, and time cost. Most volunteer coaches run on notebooks, group texts, and shared spreadsheets. The kids whose careers these coaches are shaping end up with no durable record — paper disappears, spreadsheets die with coaches, group chats roll off.

**Product:** Chalk consolidates everything a coach actually opens on their phone during a season. Five roles (admin, coach, parent, player, recruiter), multi-team by default, sport-neutral primitives where possible. Every kid gets a unique portable identifier that follows them from U8 through senior night. Every training rep, every combine number, every coach note accumulates into a structured per-player career record — visible to the coach and the parent today, mintable as an authenticated season-lock checkpoint tomorrow.

**Status (honest):** Core product is shipped and running on a real team. Combine AI analyzers for the four standard drills (vertical jump, broad jump, 5-10-5 shuttle, 40-yard dash with 10-yard split) are built; vertical and broad are being iterated on real footage, shuttle and 40 are pending first-footage validation. Claude-style LLM coaching layer, cross-league federation, NFT season-lock export, and autonomous gameday stat generation are all on the roadmap but explicitly not shipped.

**Audience:** Developers, contributors, technical evaluators. League administrators evaluating whether they can run this for their own league. Funders and grant reviewers wanting an honest technical picture.

**License:** AGPL-3.0. Anyone can self-host. Modifications to hosted versions must flow back to the community.

---

## Origin Story

This project started because a youth football team went winless, and a coach decided that wasn't good enough.

The coach is a veteran, a dad, and a volunteer. His kids played through an 0-and-everything season. They're not the next Tom Brady. They just try hard, and watching that season end the way it did didn't sit right. What kept surfacing was that the coaches, parents, and kids all deserved tools that at least gave the effort its due. One place to find things. One place to track progress. One place where a kid's eight years of youth sports actually added up to something real when he walked off the field for the last time.

Around the same time, while watching how information moved around a neighboring youth league — schedules one place, registration another, standings and communication spread across at least four separate surfaces — the same coach noticed how much harder that made every volunteer admin's job, and every family's job too. Nobody had built the tool he wished he'd had, and nobody had built the tool his league neighbors wished they'd had. So he started.

What began as a coaching manual for one team grew into a platform. The gift-economy ethos — self-hostable, no SaaS bill between you and your roster, no minors' data getting monetized — emerged directly from that origin. Nobody builds something *for the kids* while sending their data to a vendor.

This document describes what grew out of that starting point.

---

## The Problem Space

The tools already serving youth coaches fall into three camps, and each has a gap:

**Pro-tier tools (Hudl, scouting-dedicated software):** powerful, but priced and complex for college and professional programs. A volunteer youth coach running an 8U team cannot realistically afford Hudl, and would not use 80% of the features if they could. These tools optimize for deep film study and multi-analyst collaboration — not for a parent-coach tracking twelve kids with a clipboard.

**Consumer SaaS (TeamSnap, SportsEngine, GameChanger):** friendlier, but built around scheduling, messaging, and scorekeeping — not development. They store activity, not growth. A kid's eight-year journey is visible only as a chronological feed, not as a career. And the data is captured behind a vendor paywall — if the league stops paying, the data effectively stops existing.

**Notebooks, group texts, spreadsheets:** the honest default for volunteers. Works for one coach for one season. Dies the moment that coach moves on, the phone is replaced, the laptop crashes, or the kid ages up to a new team. No continuity.

What all three miss: **the kid's career as a first-class entity.** A kid who plays eight years of youth football walks away, in almost every case, with nothing permanent to show for it. The coaches who saw him improve are scattered. The numbers he put up are gone. The coaches of the *next* team he joins know nothing about who he was before he walked in.

Chalk starts from a different primitive: **the athlete persists, and the scaffolding (teams, seasons, phases, coaches) rotates around them.** Every structured data point recorded about a kid — skill ratings, training reps, combine numbers, game stats, pose observations, coach notes — accumulates against a stable athlete identity, not against a team that will disband or a season that will end. That accumulation is the career record the existing tools do not produce.

---

## What's Built Today

The scope summary below reflects shipped features as of April 2026. The [BUILD_PLAN.md](BUILD_PLAN.md) file in the repository has exhaustive per-feature detail; the [README.md](README.md) has the user-facing shipped list. This section describes what a league running Chalk today actually gets.

### Five-role architecture

Every account holds one or more of: **admin**, **coach**, **parent**, **player**, **recruiter**. A single person can hold multiple roles simultaneously (a coach who is also a parent of a player on his own team, an admin who still coaches, a recruiter evaluating his own former players). The dev-only role switcher allows previewing any dashboard without re-logging in. Role logic is applied server-side on every request — the UI never controls access.

### Multi-team foundation

Every player-scoped record (stats, combine results, attendance, training logs, scouting reports) carries a `teamId` auto-stamped from the player. Users carry `teamIds[]`. A single install serves an entire league: admin sees everything, coaches see their teams, parents see their kids. A navbar team picker scopes every page's view. Idempotent migration script lets existing installs upgrade without data loss.

### Video-verified Combine AI

Four combine analyzers shipped: vertical jump, broad jump, 5-10-5 shuttle, and 40-yard dash with 10-yard split as a sub-metric. Coach points a phone camera at the athlete running the drill, leaves the value field blank, attaches the clip on submit. The backend runs MoveNet pose inference frame-by-frame, identifies takeoff/apex/turnaround/finish events from trajectory analysis, cross-checks against coach-tapped scene calibration cones, and populates the measurement. Every value carries an AI confidence band and the original clip for coach review.

Vertical and broad jump analyzers have been field-tested with real footage and are being iterated on. Shuttle and 40-yard analyzers are built but pending first-footage validation.

The coach-friendly frame-by-frame video review modal supports speed playback (0.25x, 0.5x, 1x), jump-to-standing and jump-to-peak shortcuts driven by the analyzer's detected frames, and keyboard navigation for desktop review. When a coach disagrees with the AI measurement, the override flow requires a written reason — the trust layer is built into the UX.

### Pro Scout pose analysis

Single-athlete MoveNet pipeline for any clip. The analyzer extracts per-frame pose landmarks, identifies load-frame (deepest knee bend) and stance-frame (most upright) moments, and computes biomechanical metrics: knee bend depth, hip symmetry, posture angle — all scored against age-banded ideal ranges. Results render as a gold-skeleton overlay on extracted snapshots so coach and athlete can see exactly what the AI saw. Frontal vs. lateral view orientation is detected automatically; posture lean is silently dropped on frontal views where the measurement is meaningless.

All biomechanical thresholds live in a single constants file, editable without touching pipeline code — coaches and trainers can retune the age bands for their population.

### Training Log + Phases + Day Notes

Daily training log at **Team → Training Log**: coach picks which drills happened today, the screen renders a grid with one row per opted-in athlete and sub-columns per drill (Include / Reps / Effort 1-3 / Attitude 1-3). Sticky headers, horizontal and vertical scroll for long rosters. Opt-in model — not every kid on the roster does extra training, so parents and admins flip individual players into the program via the Manage Roster screen.

Entries auto-mark attendance present for every player in the batch. A "Present, didn't train" toggle covers sidelined players in one tap.

**Training Phases** is the periodization layer above the log. A team defines its own phases — combine prep, preseason, in-season, postseason, offseason, skills camp — each with a start date, optional end date, and active/complete status. One phase is active per team at a time; log entries auto-tag with the active phase so aggregates never conflate combine-prep reps with in-season conditioning reps. Phases are sport-neutral — basketball pre-tournament, swim taper, cheer pre-competition all fit the same primitive.

**Day Notes** capture free-form per-player-per-date context: "sick today," "teething," "rough week at home," "stayed after to work on footwork." Stored on the Attendance row, surfaced on the Training Log entry grid AND on the player's profile training history. One note, two views. Effort and attitude numbers mean more when their context is visible.

### Stats system

Per-player skill ratings on a 1-3 scale across six dimensions (Stance, Footwork, Catching, Ball Security, Plays, Effort). Per-game counter stats across ~30 offensive, defensive, and special-teams categories. Four chart views (table, radar, progress trend, season compare), plus a fifth Training Log tab showing per-player season-scoped aggregates with traffic-light coloring on effort and attitude averages. Per-player NFL-style player cards at a dedicated endpoint ready for third-party consumption.

### Game Day view

On-field scoreboard with live TD/FG/2PT/XP buttons, quarter selector, lineup builder across offense/defense/special teams, an SVG football field with draggable position-labeled player chits, a quick play log, and three-pane coach's notes (pregame/halftime/postgame). Per-game state persists to local storage and autosaves on every edit. Touch-enabled for on-field phone use.

### Scouting Workspace

Recruiter CRM with browse filters (team, age, watchlist, minimum rating), a per-player report panel with versioned 1-10 ratings across seasons, and append-only timestamped notes tagged by source (game, practice, film, combine, interview, intangibles). The substrate for the future D1-grade aggregate export that will join scout notes with game stats, combine results, and Pro Scout biomechanics into a single shareable report.

### Coach CMS (Training Guides, Drill Library, Practice Plans)

All coach-facing content is database-backed and UI-editable — no source-file edits to change a drill description. Coach CMS pattern: coaches author drafts, admin publishes, published items mutate only via an approved edit-request workflow. Drill Library supports tags, photos, safety age bands, SVG setup diagrams, and pseudo-markdown body. Practice Plan Builder arranges drills and training guides into an ordered session. Attendance Tracker handles the present/late/excused/absent roll.

### Unique Athlete Identifier (UAID)

Every player receives a portable lifetime identifier — format `LGN-2026-A3F2` (prefix configurable per-deployment). Auto-assigned on first save, backfilled across existing records on startup. CSV exports make UAID the first column; CSV imports make UAID the primary match key, preserving identity across data moves. The UAID persists across team transfers, season changes, coach rotations, and eventually league transfers. Every architectural decision downstream — trust-layer verification, NFT season-lock minting, cross-league portability, privacy-preserving LLM training data — plugs into UAID as the stable handle.

### Season Lock

A completed season can be locked. Once locked, writes to stats and games are rejected for that season — it becomes an immutable historical record. This is the checkpoint a third-party minter (NFT or otherwise) can rely on for tamper-proof career documentation.

### Admin operations

Approvals queue, user management, team management, season management, roster management with bulk CSV import/export, signup allowlist configuration, credential file viewer (admin can eyeball uploaded coaching certifications inline from the approval queue). Every admin surface is UI-friendly — the principle is that content and rosters must be editable by non-developers.

---

## Technical Architecture

### Stack

- **Frontend:** React via Create React App, React Router v7 for URL-driven routing. Inline styles + CSS variables. SVG charts (no chart library). Canvas for pose overlays and route maps. Oswald + Source Serif 4 (Google Fonts).
- **Backend:** Node.js + Express on port 5000, JWT authentication (bcrypt passwords), `pending | approved` account gate with email verification.
- **Database:** MongoDB (Mongoose) — but the Data Access Layer (DAL) abstracts every collection behind a repository interface, with Mongo adapters today and pluggable CSV/Excel adapters already in use for roster import. The DB is swap-ready.
- **Video / AI:** Multer upload, ffmpeg frame extraction, MoveNet (TensorFlow.js) pose inference, per-drill analyzer dispatcher with pure-function analyzers.

### The Data Access Layer

Every model has three layers: the Mongoose schema, the abstract repository contract (`backend/dal/repositories/`), and the concrete adapter (`backend/dal/adapters/mongo*Repository.js`). Route handlers never touch Mongoose directly — they call repository methods. This pattern does three things:

1. Swaps (e.g., to a self-hosted Postgres, to a flat-file backend, to a Cloudflare D1 adapter) are localized to a single file.
2. Testing becomes possible without spinning up Mongo — a mock repository satisfies the contract.
3. Import paths (the CSV roster importer) use the repository, so any future data source inherits the validation and side effects (auto-stamped teamId, UAID generation, etc.) for free.

### Per-drill combine analyzer dispatcher

The combine AI pipeline is built around a dispatcher at `backend/logic/drillAnalyzers/index.js`. Drill keys from the frontend constants file route to analyzer functions — `vertical_jump`, `broad_jump`, `shuttle_5_10_5`, `forty_yard` — each a pure function of the pose trajectory plus calibration metadata, returning `{ value, confidence, details }`. Adding a new drill (L-drill, 3-cone, hand timing) is a dispatcher key + analyzer file, no changes to the processor or route layer.

The processor (`backend/logic/analysisProcessor.js`) runs the pose pipeline, calls the dispatcher if a drill has a registered analyzer, and writes results back to `CombineResult` BEFORE flipping `AnalysisJob.status` to complete — this fixes a race where the frontend's poll would see "complete" before the measurement existed.

### Sport-neutral primitives

Three primitives are deliberately sport-agnostic:

- **TrainingLogEntry** — `{date, teamId, phaseId, playerId, drillId, reps, effort (1-3), attitude (1-3), notes}`. Football, basketball, soccer, cheer, swim, martial arts — all fit this shape.
- **TrainingPhase** — `{name, type, startDate, endDate, teamId, isActive}`. Every sport periodizes.
- **Drill catalog** — sport-specific content is authored per-deployment via the Coach CMS.

When the platform expands beyond youth football (the primary target today), these primitives do not fork. A soccer-club coach publishes their own drills in the Drill catalog; the training log works identically.

### Data-as-dataset discipline

Every per-player input surface uses structured, typed fields by default — numeric, enum, or reference. Free-form text is a companion to structured fields, never a replacement. Effort and attitude are integer 1-3; reps is an integer count; combine values are numeric with typed units; notes are HTML-clean plain text with original HTML preserved only for rich-text coach manual content.

This is deliberate. A future AI layer — whether Claude via API or a self-hosted fine-tune — consumes this data directly. A row reading `{reps: 10, effort: 3, attitude: 2, phaseId: <combine-prep>, date: <W2>, notes: "sick, came anyway"}` is immediately useful as model input. A row with `effort: "solid 80%"` is not.

Every new data surface is designed to survive as training data from day one, even if the model that will consume it does not exist yet.

### BYO-LLM swap-ready

The platform is explicitly designed so the AI layer can swap between Claude (retail API today), a self-hosted Llama fine-tune (medium-term), or an on-device model (long-term) without rewriting the features that consume it. When the LLM layer lands, it will follow the same DAL pattern — an abstract `LLMProvider` contract with a concrete adapter per provider — so the swap is one config change.

This is not a present-day feature. It is a present-day architectural constraint. The cost is nothing; the benefit is the ability to move off a paid API when the dataset and model maturity justify it.

### UAID as a portable identity primitive

Every Player record carries a UAID — a league-prefixed, year-stamped, random-suffixed identifier that persists for life. The generator is collision-safe with automatic retry. CSV round-trips preserve and prioritize UAID. Admin views surface UAID alongside name. The pre-save hook is `isNew`-guarded so existing UAIDs never regenerate — the ID is stable.

Downstream features consume UAID without new schema work: trust-layer parent claims key on UAID, NFT season-lock mints key on UAID, scout reports follow UAIDs across contexts, privacy-preserving ML training data keys on UAID rather than name/DOB. The decision to add UAID now, before any of those features exist, was deliberate — retrofitting identity across live data later is painful; adding an index early is free.

### No SaaS backbone

The entire architecture fits on one install. One Node process, one Mongo, one filesystem for uploads, ffmpeg in PATH. A volunteer admin can run it on a laptop on their desk, a Raspberry Pi in their closet, or a $5/month droplet. There is no background cron-as-a-service, no lambda-style component, no third-party identity vendor, no required SaaS dependency. Email verification degrades gracefully to console-log output when no SMTP is configured.

---

## Design Principles

These are not aspirational. Each one has been followed or violated in specific code decisions, and when violated, the memory record captures why.

1. **For the kids, not the SaaS bill.** The target user is a volunteer — a dad who coaches, a teacher who runs the league on weekends, a retired ref who maintains the standings. The platform must not create a subscription bill between that volunteer and the kids they serve.

2. **Gift-economy, volunteer-self-hostable.** One install. One set of instructions. No mandatory cloud, no mandatory third-party identity provider, no mandatory billing provider. AGPL-3.0 licensing insulates against a funded competitor closed-sourcing and monetizing the work.

3. **Sport-neutral where possible.** Primitives (training log, phases, drill catalog, attendance, UAID) are sport-agnostic by design. Football-specific content (stat categories, drills) lives in configurable catalogs. When the platform eventually hosts a soccer club or basketball league, the data model does not fork.

4. **Data belongs to the league.** Every export path produces usable, portable data. Every deployment owns its MongoDB. UAID makes portability practical, not theoretical. No vendor holds hostage data.

5. **Structured numeric signal + free-form companion.** Primary data fields are numeric or enum. Free-form notes exist where human context matters but are never the only carrier of a signal.

6. **The player is permanent. Everything else is scaffolding.** Teams rename, coaches rotate, seasons end, phases complete, admins change. The athlete — their UAID, their accumulated record — persists. Every design tradeoff resolves in favor of player continuity.

### A note on adjacent domains

The primitives above — rostered people, per-person-per-activity structured logs, periodization phases, permanent individual identity — are domain-neutral by design. Youth sports is the primary application, but the same architecture fits school districts, after-school programs, physical therapy clinics, corporate training, dance studios, martial arts programs, and volunteer organizations. This document describes the youth-sports application; the architecture does not preclude others. We are not pursuing adjacent markets during the current priority window. We are noting, honestly, that the architecture generalizes.

---

## Trust Layer

The platform holds minors' data. That imposes obligations — legal (COPPA, BIPA, state-level privacy laws), ethical (kids' data should not be a product), and practical (parents will not trust what is not trustworthy).

The trust layer is a staged roadmap, not a single feature:

**Tier 1 — Parental controls that must exist before any public-facing launch:**
- Media and biometric consent: per-video, per-clip opt-in before pose analysis or video storage
- Scout / recruiter access gate: parent decides if recruiters can see their kid at all, defaulting to opt-in for 13+ and opt-out for under-13
- Leaderboard visibility toggle: parent controls whether their kid appears on combine leaderboards
- Data export: comprehensive per-kid data dump on parent request
- Data delete: COPPA right to have a kid's record removed
- Correction request: parent can flag incorrect data for admin fix

**Tier 2 — Landing with verified credentials:**
- Verified parent badges — parent-kid link is either self-claimed (unverified) or roster-matched (verified). Only verified parents can flip sensitive toggles.
- Two-parent co-sign for sensitive changes (moving between teams, profile going public) when a player has multiple parents linked
- Age-gated automatic protections: under-13 gets extra safeguards regardless of individual settings
- Communication gate: no direct messaging between a non-parent adult and a kid routes around the parent dashboard

**Tier 3 — Advanced:**
- Per-recruiter time-boxed access
- Parent-controlled audit log ("who has looked at my kid's profile this month")
- Delegation to secondary guardians
- Scheduled auto-revoke of any permission

**Season Lock** is the immutability checkpoint. Once a season is locked, the stats and games within it are frozen — the record is tamper-proof. This is the substrate a third-party minter (NFT or otherwise) can rely on for authenticated lifetime career documentation.

The trust layer is work in progress. It lands before any broad public launch; it is deferred during the current priority window where the platform is being validated against a specific single-league pilot with known parents.

---

## Roadmap

Labels below are honest about what is live, what is in progress, and what is speculative.

### Live today
Every section under **"What's Built Today"** above.

### In progress
- Shuttle and 40-yard combine analyzer field validation against real footage (filming pending)
- Vertical and broad jump iteration on kinks surfaced in field validation

### Near-term (next few months)
- **On-field game timer** and practice-plan timer with auto-advance and buzzer
- **Post-practice share-outs**: coach types raw notes after practice, system drafts a parent-friendly email summary, coach reviews and sends
- **Forms UI**: medical release, waivers, consent — upload, sign, track
- **Keypoint retention (lite)**: stop discarding MoveNet pose-keypoint sequences after inference. Accrue the dataset. No UI change, no consent change — just save the structured numeric signal for future model training.
- **Trust layer tier 1**: the parental controls listed above

### Mid-term (this year to next)
- **Claude-style LLM coaching copilot**: practice plan drafting based on player weaknesses, scouting report prose, parent share-out writing, natural-language drill library search. Runs on a swap-ready adapter so the backend can move from retail API to self-hosted Llama or an eventual custom fine-tune without feature changes.
- **League platform tools**: director-level dashboard (the commissioner dashboard), shared drill pool, league-wide schedule, cross-team standings
- **Docker-compose one-command deploy**: the gift-economy endgame made real — a volunteer admin runs `docker-compose up`, gets a working deployment
- **Self-host + bring-your-own-cloud documentation**: explicit walkthroughs for running on a home server, a $5/month droplet, Oracle Cloud Free Tier, etc.

### Long-term / speculative
- **NFT season-lock export**: locked seasons produce a signed JSON + hash that a third-party minter can consume. One UAID → one lifetime career record → a mintable thing if the athlete or league chooses.
- **Autonomous Gameday AI**: multi-camera capture, player tracking via jersey OCR, play segmentation, automatic stat assignment, per-play biomechanical observations. Film-to-box-score with no coach data entry. Built on the same pose engine Pro Scout already uses.
- **Custom on-device coaching model**: a fine-tuned model trained on accumulated training-log, combine, and correction-label data, running locally on a coach's phone. The BYO-LLM adapter pattern means the feature surface does not change; the adapter underneath does.

### Not a goal
- Becoming a SaaS. If this platform succeeds as intended, it is because many leagues run their own instances — not because one company hosts them all.

---

## How to Contribute + Self-Host

The repository is open under AGPL-3.0. Anyone can clone, run locally, modify, deploy, and run it for their own league.

**Running locally:**
```bash
git clone <repo-url> chalk
cd chalk/backend && npm install
cd ../frontend && npm install
# Ensure MongoDB is running locally + ffmpeg is on PATH
# Set JWT_SECRET in backend/.env
cd ../backend && node server.js        # Terminal 1: backend on :5000
cd ../frontend && npm start             # Terminal 2: frontend on :3000
```

**Deploying for a real league:**
- Backend: Render, Fly.io, Railway, or any $5/month VPS with Node + ffmpeg
- Database: MongoDB Atlas free tier (512MB, generous for a league)
- Frontend: bundled with backend (serve static from Express) or separately on Render/Netlify/Cloudflare Pages
- HTTPS: automatic on most cloud providers, free via Let's Encrypt if self-hosting

**Contributing:**
- Issues and PRs welcome in areas marked *in progress* or *near-term* above
- For architectural changes, open a discussion issue first
- For docs, typo fixes, accessibility improvements — direct PRs are welcome

**What we are not accepting:**
- SaaS-flavored additions that assume a hosted business model
- Features that introduce mandatory third-party dependencies
- Work that violates the design principles without discussion

Contributions are reviewed for fit against the principles in the previous section, not just for code quality.

---

## License + Attribution

**License:** AGPL-3.0. The strong copyleft terms are deliberate — a funded competitor cannot take the code, wrap it in a proprietary hosted product, and capture the community's work without giving modifications back. Self-hosting for your own league is unrestricted. Running it as a service for others requires making source available.

**Upstream acknowledgments:**
- **MoveNet** (Google Research, via TensorFlow.js) — single-person pose inference, the foundation of Pro Scout and the combine analyzers
- **React** (Meta) — the frontend framework
- **Express** and the Node.js ecosystem
- **MongoDB** and Mongoose
- **ffmpeg** — frame extraction and image resizing
- **react-router-dom v7** — URL-driven routing
- **csv-parse**, **bcryptjs**, **jsonwebtoken**, **nodemailer**, **multer**, **mime-types**, and the rest of the CommonJS toolchain

The platform builds on their work. Their licenses are preserved in full.

**Trademarks:** The product name is being finalized. References in the repository to "352 Legends" refer to the youth football organization the author coaches for — a league the author serves but does not own as a brand. The platform is independent of that league and is not affiliated with, endorsed by, or representing any specific youth sports organization.

**Not affiliated with:** Hudl, TeamSnap, SportsEngine, GameChanger, USA Football, or any of the incumbent youth sports platforms. The architecture was developed independently from first principles.

---

## Appendix A: Key Files Reference

For developers exploring the codebase:

- `backend/server.js` — route mounting, migrations, analysis processor loop
- `backend/models/` — Mongoose schemas for every collection
- `backend/dal/` — Data Access Layer contracts and Mongo adapters
- `backend/routes/` — Express routers per feature area
- `backend/logic/drillAnalyzers/` — per-drill combine analyzer pure functions
- `backend/logic/analysisProcessor.js` — video analysis job queue worker
- `backend/logic/poseInference.js` — MoveNet pipeline wrapper
- `backend/utils/uaid.js` — Unique Athlete Identifier generator
- `frontend/src/App.js` — React Router tree
- `frontend/src/components/dashboards/` — the five role-specific dashboards
- `frontend/src/components/combine/CombineEntry.js` — combine AI entry workspace
- `frontend/src/components/trainingLog/TrainingLog.js` — daily training log entry grid
- `frontend/src/components/shared/` — Navbar, AppShell, TeamPicker, ProtectedRoute

The [BUILD_PLAN.md](BUILD_PLAN.md) document in the repository is the tactical per-feature plan; this whitepaper is the strategic vision. They cross-reference.

---

## Appendix B: Design Principle Memories

The platform's design principles are captured as persistent memory that shapes every future decision:

- Sport-neutral primitives; don't fork per sport
- Data-as-dataset; structured + numeric by default
- UI-editable content (all coach/admin CRUD)
- Rich-text authoring for long-form content (not markdown syntax)
- AI override requires written reason (protects trust layer)
- Design-first principles: CSS vars only, inline styles, no hex in components, ROLES as constants
- Mullet architecture: opinionated UX over admin-editable primitives

Code follows these; drift gets caught in review.

---

## Closing Note

This platform exists because a dad got tired of watching kids' effort disappear into the void of end-of-season scorebooks. The code is the part that is testable. The reason it is the code is what the code tries to preserve.

Every kid who steps on a field and puts in the kind of effort I've watched these last two seasons deserves a record of what they did. Every volunteer coach who shows up three nights a week deserves tools that treat their work like it matters. Every parent deserves to see what their kid is becoming, not just their game-day numbers — and deserves a tool that puts them in the same tier as the pros, for free.

That is what Chalk is trying to be.

The architecture above is how. The code is where. This document will update as the platform does.

---

*Chalk is open source under AGPL-3.0. Build, break, fork, improve, contribute back.*

*Built for the kids.*
