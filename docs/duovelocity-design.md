# DuoVelocity — Design Doc (v1)

**Status:** Draft · 2026-09-15
**Owner:** David Sopko
**Working name:** DuoVelocity (placeholder)

---

## 1. Purpose

Duolingo shows you a streak and today's XP, but nothing about *pace*. DuoVelocity captures a user's Duolingo progress every night and turns it into velocity: how many lessons per day, how many units per week and per month, and how the Duolingo Score — what most users call their level — moves over time.

The core problem is that Duolingo keeps per-lesson history for only ~1–2 weeks (14 days observed) and stamps no completion dates on the learning path at all. So DuoVelocity's job is to **observe daily and remember**: archive lesson events before they roll off, and manufacture unit/node completion dates by diffing nightly snapshots.

**DuoVelocity tracks exactly four things.** Everything else in the payload is either supporting data or noise.

1. **`LESSON` events** — path-advancing lesson events (`xpGains[]` with `eventType = LESSON`), each carrying a `time` and a `skillId`. These are the timestamped record of what was done and when, and they must be archived before the ~1–2 week window erases them (§7.1). They drive lessons/day.
2. **Units** — a unit's completion is detected by diffing nightly snapshots and dated from the finishing `LESSON` event (§9.3). Units drive units/week and units/month. Nodes exist only to detect and date unit completion.
3. **Score** — the Duolingo Score, `currentCourse.scoreMetadata.reachedScore` (§7.2), recorded over time.
4. **Daily XP** — XP earned per day, from the `xp_summaries` endpoint. Cheap to get and available ~15 months back, so it needs no forward-only archiving. It gives the user a rough read on **consistency** — how often and how much they show up. It is a daily total, not lesson-level, so it does not distinguish lessons from other activity.

Deliberately outside this: lifetime XP-total dashboards, streak, and leagues, all of which Duolingo already shows. (Daily XP *is* tracked — as a consistency signal over time, not a running total.)

## 2. Goals and non-goals

**Goals (v1)**
- Multi-user from day one: anyone can sign up, connect their Duolingo account, and see their own velocity.
- Nightly sync at 04:30 US Eastern, per user, with no gaps larger than the `xpGains` window (~1–2 weeks; 14 days observed).
- Metrics, all derived from the four tracked things (§1): lessons/day from `LESSON` events, units/week and units/month from unit completions, Score over time, and daily XP for a consistency read. Nodes are supporting data (how unit completion is detected and dated), not a v1 metric.
- A bearer-token JSON API that any frontend (web or mobile) can consume. Frontend itself is TBD and out of scope for this doc.
- $0/month hosting at hobby scale on Azure free tiers.
- Honest credential handling: users are told plainly what DuoVelocity holds and what the operator can see.

**Non-goals (v1)**
- Streak dashboards or lifetime XP-total dashboards (Duolingo already does these). Daily XP *is* tracked, but as a per-day consistency signal, not a running total.
- Vocabulary tracking, leagues, friends.
- Multiple courses per user — v1 tracks the user's *current* course only.
- Historical backfill beyond what the API exposes at connect time.

## 3. Vocabulary

These six words are the entity names in `Core`. The word **"level"** is banned — it means three different things in Duolingo-land (a path node in the API, the old XP-based 1–25 level, and what users call the Score).

| Term | Meaning | API source |
|---|---|---|
| **Section** | Group of units (e.g. "Section 5") | `currentCourse.pathSectioned[]` |
| **Unit** | Titled group of nodes (e.g. "Unit 63: Say where you want to go") | `pathSectioned[].units[]` |
| **Node** | One bubble on the path. Observed `type`s: `skill`, `story`, `practice`, `duo_radio`, `chest`, `unit_review` | `units[].levels[]` (`type`, `state`, `finishedSessions`, `totalSessions`) |
| **Lesson** | One session inside a lesson node. A node holds *N* lessons (`totalSessions`); N may be 1. | `xpGains[]` where `eventType == LESSON` |
| **Activity** | XP-earning work *off* the path (practice zone, review of a completed node, stories replayed) | `xpGains[]` where `eventType != LESSON` |
| **Score** | The **Duolingo Score**, shown top-left on the path and on the profile: a CEFR-aligned proficiency number, **0–130** for the major courses (English, Spanish, French), capped lower on less-developed courses. Rises as lessons, units, and sections are completed. This is what users (and older UI copy) call their "level"; named Score internally. Distinct from XP — see §7.2 | `currentCourse.scoreMetadata.reachedScore` (M0) |

## 4. Metric definitions

All "days" are the **user's** days, in the user's Duolingo timezone.

| Metric | Definition |
|---|---|
| Lessons/day | Count of `XpEvent` rows with `EventType = 'LESSON'`, grouped by `EventLocalDate` |
| Activities/day | Same, `EventType <> 'LESSON'` — shown separately, never mixed in |
| Units/week | Count of `UnitCompletion` rows grouped by ISO week of `CompletedLocalDate` |
| Units/month | Count of `UnitCompletion` rows grouped by month of `CompletedLocalDate` |
| Score trend | `ScoreHistory` by date |
| XP/day | `DailyXp.GainedXp` by `LocalDate` — the consistency read |
| Summary | 7-day and 30-day rolling averages of lessons/day and XP/day; last unit completed; last sync |

Nodes (`NodeCompletion`) are recorded but not surfaced as a metric in v1 — they exist to derive unit completions and to attribute timestamps.

Design rule: **never assume the lesson-to-node ratio.** Lessons come from events; nodes come from the snapshot diff. If Duolingo has made it 1:1 in a course, the numbers agree on their own.

## 5. Architecture

```
                 ┌──────────────────────────────────────────────┐
                 │               DuoVelocity.Core               │
                 │  IDuolingoClient · ISyncService · IMetrics   │
                 │  IConnectionService · storage (Azure SQL)    │
                 └───────▲──────────────▲──────────────▲────────┘
                         │              │              │
        ┌────────────────┴───┐  ┌───────┴────────┐  ┌──┴───────────────────┐
        │  DuoVelocity.Cli   │  │ .Functions     │  │  DuoVelocity.Api     │
        │  sync / inspect /  │  │ Timer 04:30 ET │  │  ASP.NET Core Web    │
        │  decode-token /    │  │  → queue fan-  │  │  API, bearer auth,   │
        │  metrics           │  │  out → SyncUser│  │  App Service F1      │
        └────────────────────┘  └────────────────┘  └──────────▲───────────┘
                                                               │
                                                     Frontend (TBD: web/mobile)
```

Principles:
- `Core` is the application. It knows nothing about how it is hosted. All logic, all `ILogger<T>` logging, all `IOptions<T>` config lives here.
- Every host is thin. The Function's timer body is essentially `await enqueue.RunAsync()`; the queue handler is `await sync.RunAsync(userId)`. The CLI and API call the same services.
- The nightly sync runs **in-process** in the Function (it references `Core` directly). It never calls the API over HTTP, so it does not care whether the F1 App Service is asleep.
- The API is a read surface plus the connect/disconnect flow. It owns no business logic of its own.

## 6. Solution structure

```
DuoVelocity.sln
├── src/
│   ├── DuoVelocity.Core/            class library (.NET 10)
│   │   ├── Duolingo/                IDuolingoClient, DTOs, JwtInspector
│   │   ├── Sync/                    ISyncService, PathDiffer, EventIngester
│   │   ├── Connections/             IConnectionService, ICredentialProtector
│   │   ├── Metrics/                 IMetricsService
│   │   └── Data/                    DbContext or Dapper repos, migrations
│   ├── DuoVelocity.Cli/             System.CommandLine host
│   ├── DuoVelocity.Functions/       .NET isolated worker, Windows Consumption
│   └── DuoVelocity.Api/             ASP.NET Core Web API (.NET 10)
├── tests/
│   └── DuoVelocity.Core.Tests/      diff algorithm, parsers (fixture JSON), tz bucketing
├── db/                              schema + migrations
├── docs/
│   ├── duovelocity-design.md        this document
│   └── duolingo-api.md              endpoint/field reference (see §7)
└── samples/                         redacted raw JSON fixtures (no tokens, no PII)
```

Storage access: EF Core or Dapper — either fits; Dapper is the lighter choice for a schema this small and matches how the queries will be written anyway. Decide at M1.

## 7. Data source — Duolingo unofficial API

Duolingo publishes no API. DuoVelocity calls the same endpoints the web client uses. This is the single largest risk in the project (§14) and shapes several design choices. The full endpoint and field reference lives in **`docs/duolingo-api.md`**; this section covers only what the design depends on.

**Auth:** the JWT is the credential. It is sent as `Authorization: Bearer`. Login (`POST /2023-05-23/login`, body `{ identifier, password, signal }`) requires a reCAPTCHA Enterprise token in `signal`, which only a real browser can mint — see §10.3 and §14 risk #1. So DuoVelocity does not log in server-side; the user supplies a token captured from a signed-in browser session. M0 confirmed the token lifetime: the `exp` claim is set ~200 years out and `iat` is unset, so the token does not self-expire. Only a Duolingo password change or a server-side revocation invalidates it (14-day empirical revocation test running as of 2026-09-18).

**Primary endpoint:** `GET /2017-06-30/users/{id}?fields=...` (authenticated) with an explicit `fields` list. Fields used:

| Field | Used for |
|---|---|
| `id`, `username`, `timezone`, `learningLanguage`, `fromLanguage` | identity, day bucketing |
| `courses[]` (`id`, `title`, `learningLanguage`, `fromLanguage`, `xp`, `crowns`) | course registry; which is current |
| `xpGains[]` (`xp`, `skillId`, `time`, `eventType`) | lesson and activity events. `time` is the only per-lesson timestamp (Unix epoch seconds). **~1–2 week retention (14 days observed)** — this is why sync is nightly and why a longer gap is an irrecoverable loss. M0 saw `eventType` values `LESSON`, `PRACTICE`, and `null` — see §7.1 |
| `currentCourse.pathSectioned[]` → `units[]` → `levels[]` (`type`, `state`, `finishedSessions`, `totalSessions`, `pathLevelMetadata`/`pathLevelClientData` incl. skill ids, unit index/name) | node and unit state. **No timestamps — confirmed by M0** (§7.1); completion dates are derived by diffing |
| `currentCourse.scoreMetadata` (`reachedScore`, `pathStartingScore`, `pathEndingScore`, `supportType`) | the Duolingo Score and this course's Score range and cap — M0 resolved, see §7.2 |

**Landing rule:** every response is stored verbatim (`RawSnapshot`) before parsing. Field names drift; the raw column is the insurance policy that lets history be re-parsed later.

**Etiquette:** a small number of requests per user per night (the user payload, plus `xp_summaries`, plus the path — ~575 KB gzipped, ~9 MB parsed — only when new lessons warrant it, §9.2), sequential with a small delay, a descriptive `User-Agent`, exponential backoff on 429/5xx. Never hammer.

### 7.1 Timestamps in the payload (M0 findings, 2026-09-18)

M0 read a live payload and searched every field for anything temporal. The result confirms the project's founding premise: **the learning path carries no completion dates, so the only per-lesson timestamp is `xpGains[].time`, and it lives for only ~1–2 weeks (14 days observed).** Everything else is either account-level or streak-level and daily-resolution at best.

**The one timestamp that matters — `xpGains[].time`.** Each `xpGains` entry (a lesson or activity event) has a `time` field in Unix epoch seconds. This is the per-lesson timestamp: it is what lets §9.3 attribute a real completion time to a node (`Attribution = LessonEvent`) instead of falling back to snapshot day. It is also the field that ages out after ~1–2 weeks (14 days observed), which is the whole reason the sync must run nightly. In the sample, 74 events came back with `eventType` split `LESSON` 25, `PRACTICE` 12, and **`null` 37** — a `null` type (skill id also null) is common, so §4/§5 bucketing must treat "not `LESSON`" as activity rather than testing for a specific activity type. Log unknown/`null` types (§14 risk #5).

**The path has no dates — confirmed.** A path node (`levels[]`) exposes `state` (e.g. `"passed"`), `pathLevelMetadata.nodeState` (e.g. `"passed"`), `finishedSessions`/`totalSessions` (counts), `type`/`subtype`, skill ids, indices, and score info. **None of these is a timestamp.** Names that pattern-match as temporal — `completedUnits`, `finishedSessions`, `finishedLessons`, `finishedLevels`, `completedLevels`, `checkpointFinished`, `finalLevelTimeLimit` — are all counts, flags, or a per-lesson time *limit* in seconds, never a completion date. This is why node and unit completion dates must be manufactured by diffing nightly snapshots (§9.3), exactly as designed.

**Other timestamps exist but are not per-lesson.** They are useful context, not a completion source:

| Field | Format | What it is |
|---|---|---|
| `creationDate` | epoch seconds | account creation date (one value, historical) |
| `streakData.startTimestamp` | epoch seconds | when the current streak began |
| `streakData.updatedTimestamp` | epoch seconds | last streak update (rolls over daily) |
| `streakData.churnedStreakTimestamp` | epoch seconds | 0 when not churned |
| `streakData.updatedTimeZone` | IANA string | e.g. `America/New_York` — cross-checks the profile `timezone` used for day bucketing |
| `streakData.currentStreak` / `longestStreak` / `previousStreak` | objects of ISO `YYYY-MM-DD` date strings (`startDate`, `endDate`, `lastExtendedDate`, `achieveDate`) | streak boundaries at **day** resolution only |

Takeaway for the data model: none of these replaces the snapshot diff. `xpGains[].time` sharpens node-completion attribution while it is fresh; the streak dates are day-resolution and about streaks, not lessons or units. The `RawSnapshot` landing rule (above) still captures all of it verbatim so any of these can be re-parsed later.

### 7.2 The Duolingo Score (M0 findings, resolves §14 risk #4)

The Score is the CEFR-aligned proficiency number shown top-left on the path and on the profile — the number many users call their "level." M0 located it and read the surrounding metadata.

**Field:** `currentCourse.scoreMetadata`, an object:

| Key | Sample | Meaning |
|---|---|---|
| `reachedScore` | 66 | the user's current Score — this is the value `ScoreHistory` records |
| `pathStartingScore` | 5 | Score at the start of this course's path |
| `pathEndingScore` | 129 | this course's cap — **read it, never hardcode** |
| `supportType` | `FULLY_CEFR_ALIGNED` | how far the course is CEFR-aligned |

**Range and cap.** The Score runs 0–130 on the fully built courses (English, Spanish, French; more are being extended to 130). Less-developed courses stop lower — some at 60 or less — because the cap tracks how much CEFR-aligned content exists. So the ceiling is per-course: use `pathEndingScore`, not a constant. (The doc previously said 0–160; that was wrong.)

**CEFR bands** (from duoplanet.com/duolingo-score, corroborated by the API's own per-unit `cefrLevel` and a band string of `60-79`/`B1` observed for score 66):

| Score | CEFR |
|---|---|
| 0–9 / 10–19 / 20–29 | A1 (very early / early / high) |
| 30–59 | A2 |
| 60–79 / 80–99 | B1 (early / high) |
| 100–114 / 115–129 | B2 (early / high) |
| 130 | cap of the major courses (high B2) |

Treat these cut points as Duolingo's rule of thumb, not a contract: if a CEFR label is ever shown, read `cefrLevel` from the payload rather than computing it from the number.

**What moves it.** The Score rises as lessons, units, and sections are completed (practice, stories, and the like also contribute). In the path data this shows up as a per-node `levelScoreInfo.reachedScore` and per-section `sectionStartingScore`/`sectionEndingScore`, climbing toward `pathEndingScore`. For v1, DuoVelocity reads only the course-level `scoreMetadata.reachedScore` and records changes; it does not need the per-node score breakdown.

**Score is not XP and not streak.** Three independent numbers: Score is proficiency (0–130, hundreds of hours to max out), XP is lifetime activity points (this account: ~195k), streak is consecutive days. DuoVelocity tracks Score over time as a velocity metric and never conflates it with XP.

**Terminology.** The word "level" is banned internally (§3) precisely because this payload overloads it: a path node is a `level`, there is `levelScoreInfo`, `cefrLevel`, and `crownLevelIndex`, and users call the Score their "level." The 0–130 number is only ever `scoreMetadata.reachedScore`.

## 8. Data model

Conventions per `DB_Design_Standards.md`: PK named `<TableName>Id`; all timestamps `DATETIME2(3)` in UTC with `Utc` suffix; local dates as `DATE` with `LocalDate` suffix; audit columns `CreatedUtc` / `ModifiedUtc`.

**`AppUser`** — one row per DuoVelocity account
`AppUserId`, `ExternalSubject` (IdP subject, unique), `Email`, `DisplayName`, `TimeZoneId` (IANA, copied from Duolingo profile, nullable until first sync), `CreatedUtc`, `ModifiedUtc`

**`DuolingoConnection`** — one per user (unique on `AppUserId`)
`DuolingoConnectionId`, `AppUserId`, `DuolingoUserId`, `DuolingoUsername`, `JwtCiphertext VARBINARY(MAX)`, `JwtExpiresUtc NULL`, `KeyVersion`, `Status` (`Active` · `TokenRevoked` · `Paused` · `Disconnected`), `LastSyncUtc`, `LastSuccessUtc`, `ConsecutiveFailures`, `CreatedUtc`, `ModifiedUtc`
No password is ever stored — the token is the only credential DuoVelocity holds. `TokenRevoked` is set when Duolingo returns 401, meaning the user changed their password or the token was revoked; the user must reconnect with a fresh token.

**`Course`** — courses seen for a user
`CourseId`, `AppUserId`, `DuolingoCourseId`, `LearningLanguage`, `FromLanguage`, `Title`, `IsCurrent`, `CreatedUtc`, `ModifiedUtc`

**`RawSnapshot`** — verbatim API responses
`RawSnapshotId BIGINT`, `AppUserId`, `CapturedUtc`, `Endpoint`, `Payload VARBINARY(MAX)` (gzip-compressed JSON — see size note below; `NVARCHAR(MAX)` if stored uncompressed), `PayloadHash BINARY(32)`, `CreatedUtc`
Retention: keep all (v1). **Size reality (M0, corrected 2026-09-21):** the `currentCourse` path is ~9 MB of JSON (7,635 nodes) but gzip-compresses to **~575 KB** on the wire (the web app's own call transfers 574 KB and decodes to 8.9 MB; `HttpClient` requests gzip by default). So the cost depends on how it is stored: **store `RawSnapshot` compressed** (`VARBINARY`, gzip) and it is ~575 KB/night ≈ ~200 MB/user/year, comfortably inside the free-tier 32 GB cap (§12); store it as decompressed `NVARCHAR(MAX)` JSON and it is ~9 MB/night ≈ ~3 GB/user/year, which fills 32 GB in a few user-years. Recommendation: compress the payload column. Further mitigations: only fetch the path on nights with new `LESSON` events (§9.2) so quiet nights store nothing; keep raw only when the `PayloadHash` changed; store the diff plus current `PathNode` state rather than a verbatim path every night.

**`SyncRun`** — one row per attempt
`SyncRunId`, `AppUserId`, `StartedUtc`, `FinishedUtc`, `Outcome` (`Success` · `TokenRevoked` · `Failed`), `Message`, `XpEventsInserted`, `NodesCompleted`, `UnitsCompleted`, `CreatedUtc`

**`XpEvent`** — every `xpGains` record ever seen (lessons *and* activities)
`XpEventId BIGINT`, `AppUserId`, `CourseId`, `EventUtc`, `EventLocalDate`, `EventType`, `SkillId NULL`, `Xp`, `CreatedUtc`
Unique natural key: `(AppUserId, EventUtc, EventType, SkillId, Xp)` — Duolingo provides no row id.

**`PathNode`** — current state of every node in a course (upserted nightly; the "previous state" for the diff)
`PathNodeId`, `CourseId`, `SectionIndex`, `UnitIndex`, `UnitName`, `NodeIndex`, `NodeType`, `SkillIds NVARCHAR(400)` (JSON array), `TotalSessions`, `FinishedSessions`, `State`, `IsComplete`, `IsLegendary`, `FirstSeenUtc`, `LastSeenUtc`, `CreatedUtc`, `ModifiedUtc`
Unique: `(CourseId, SectionIndex, UnitIndex, NodeIndex)`

**`NodeCompletion`** — the manufactured timestamp
`NodeCompletionId`, `PathNodeId`, `AppUserId`, `CompletedLocalDate`, `CompletedUtc`, `Attribution` (`LessonEvent` · `SnapshotDay`), `CreatedUtc`
Unique: `(PathNodeId)` — a node completes once (legendary is a separate flag, not a second completion)

**`UnitCompletion`**
`UnitCompletionId`, `CourseId`, `AppUserId`, `SectionIndex`, `UnitIndex`, `CompletedLocalDate`, `CompletedUtc`, `CreatedUtc`
Unique: `(CourseId, SectionIndex, UnitIndex)`

**`ScoreHistory`**
`ScoreHistoryId`, `AppUserId`, `CourseId`, `LocalDate`, `Score`, `CreatedUtc`
`Score` is `currentCourse.scoreMetadata.reachedScore` (§7.2). The per-course cap `pathEndingScore` is a property of the `Course`, not a daily fact, so it belongs on `Course` if surfaced at all, not here.
Unique: `(AppUserId, CourseId, LocalDate)`

**`DailyXp`** — one row per user per active day
`DailyXpId`, `AppUserId`, `LocalDate`, `GainedXp`, `NumSessions`, `CreatedUtc`, `ModifiedUtc`
Sourced from the `xp_summaries` endpoint (`date` → `LocalDate`, `gainedXp`, `numSessions`). Unlike the other tables this is **not** limited to the sync window: `xp_summaries` reaches ~15 months back, so `DailyXp` is backfilled at connect time and topped up each sync. `NumSessions` is stored for context but is not a lessons count. Inactive days have no row (read as zero).
Unique: `(AppUserId, LocalDate)`

Why store only transitions rather than nightly node state: a course has hundreds to thousands of nodes; a per-node-per-day table would be ~700k rows per user per year with almost no information in it. `RawSnapshot` already holds the full history; `PathNode` holds "now"; `NodeCompletion` holds the change.

## 9. Nightly sync

### 9.1 Trigger and fan-out
1. **`EnqueueSyncs`** (Timer, NCRONTAB `0 30 4 * * *`, `WEBSITE_TIME_ZONE = Eastern Standard Time`) selects every `DuolingoConnection` with `Status = 'Active'` and drops one message per user on a Storage Queue. A `TokenRevoked` connection is skipped until the user reconnects.
2. **`SyncUser`** (Queue trigger, `batchSize` 1–2, low concurrency) runs `ISyncService.RunAsync(appUserId)` for one user.

Why a queue rather than a loop inside the timer: per-user isolation (one user's failure doesn't abort the batch), free retries with poison-queue handling, no risk against the Consumption plan's function timeout (default 5 min, max 10) as users grow, and a manual sync is just "drop a message."

04:30 Eastern is 01:30 Pacific — the previous day is closed for every US timezone. Non-US users get a slightly stale "yesterday"; acceptable for v1.

### 9.2 Per-user run
```
load connection → decrypt JWT
  if Duolingo returns 401:
      Status=TokenRevoked, Outcome=TokenRevoked, flag user to reconnect, STOP
      (no server-side re-login: login needs a browser-minted captcha token)
fetch light payload (fields = xpGains, courses, timezone — NOT the full path) → RawSnapshot
upsert Course rows; ensure TimeZoneId on AppUser
ingest xpGains → XpEvent (skip natural-key duplicates)
fetch xp_summaries (startDate = last stored DailyXp date, or full ~15mo on first sync) → upsert DailyXp
if new LESSON events since last successful sync:          # skip the path pull on quiet nights
    fetch currentCourse (path + scoreMetadata, ~575 KB gzipped / ~9 MB parsed) → RawSnapshot
    diff pathSectioned against PathNode → NodeCompletion, UnitCompletion; upsert PathNode
    extract Score → ScoreHistory (if changed or first of day)
write SyncRun; update LastSyncUtc / LastSuccessUtc / ConsecutiveFailures
```
The `currentCourse` pull is the heaviest step (~575 KB gzipped, ~9 MB parsed), so it runs only when `xpGains` shows lessons since the last sync — no new lessons means no unit progress and no path fetch. Score rides along with that pull (it lives inside `currentCourse`); a lighter Score-only source exists (`score-info`, API doc §11.3) but its exact request is unconfirmed, so on quiet nights Score is simply left unchanged. Every step is idempotent — re-running the same night inserts nothing new. `Duolingo` HTTP failures retry with backoff inside the run; anything still failing goes back to the queue (max 3 deliveries) then to poison.

### 9.3 Node completion and timestamp attribution
A node is **complete** when `state` is `passed` (or `legendary`), which is the authority. `finishedSessions == totalSessions` usually agrees, but M0 found a `unit_review` node reading `state: passed` with `finishedSessions: 0`, so **trust `state`, not the session counts**, and log any disagreement.

Observed node `type` values (M0): `skill`, `story`, `practice`, `duo_radio`, `chest`, `unit_review`. The list is open — log unknowns (§14 risk #5).

When tonight's payload shows a node complete that `PathNode` had as incomplete:
1. Look for `XpEvent` rows with `EventType = 'LESSON'`, a `SkillId` in the node's `SkillIds`, and `EventUtc` after the previous successful sync. If found, `CompletedUtc` = the latest such event, `Attribution = LessonEvent`. This turns a day-resolution guess into a real timestamp.
2. Otherwise `CompletedLocalDate` = yesterday (user's tz), `CompletedUtc` = end of that local day, `Attribution = SnapshotDay`.

A **unit** is complete when every progression node in it is complete. Progression nodes are the `skill`, `story`, `practice`, `duo_radio`, and `unit_review` types; **`chest` does not count**. The unit's terminal node is the `unit_review`, identifiable directly by `levelScoreInfo.touchPointType == "UNIT_END"` (every other node reads `NORMAL`); it also carries `pathLevelMetadata.unitIndex`. Unit timestamp = the latest `NodeCompletion` among its progression nodes.

Legendary: flip `IsLegendary`, do not create a second completion.

### 9.4 Gaps
`xpGains` holds ~1–2 weeks (14 days observed; treat the window as variable and sync well inside it). If a user's last success is older than that, lessons in the gap are lost and the run logs a `GapDetected` warning on `SyncRun`. Two consecutive failures raise an alert (§13). Node/unit completions are *not* lost by a gap — the diff still catches them, they just fall back to `SnapshotDay` attribution.

## 10. Identity, tenancy, credentials

### 10.1 App identity
Users sign in to DuoVelocity through an external identity provider; DuoVelocity never stores a DuoVelocity password. Recommended: **Microsoft Entra External ID** (social + email sign-in, free for the first 50k monthly active users — verify current pricing at build time). The API validates bearer tokens with `Microsoft.Identity.Web`; `AppUser.ExternalSubject` is the IdP subject claim. Any frontend — SPA, Blazor, or a mobile app — gets a token from the IdP and calls the API. Nothing in the API assumes a browser.

### 10.2 Tenancy
Every table with user data carries `AppUserId`, and every repository method takes the caller's `AppUserId` from the validated token — never from the request body. There is no cross-user query surface in v1.

### 10.3 Duolingo credentials — the honest version
The unofficial API needs a real login, and M0 confirmed that login is captcha-gated: the `POST /2023-05-23/login` request carries a reCAPTCHA Enterprise token that only a real browser can produce. A headless Azure Function cannot mint one, so DuoVelocity never logs in on the server and never holds a Duolingo password. **The token is the credential, full stop.** Design:

- **The token, only.** On connect, the user pastes a JWT captured from their signed-in Duolingo browser session (a short guide shows where DevTools exposes the `jwt_token` cookie). The API validates it with one authenticated call, then stores it encrypted. There is no username/password field.
- **No expiry, so no re-login loop.** M0 found the token's `exp` claim is ~200 years out with `iat` unset — it does not self-expire. It dies only when the user changes their Duolingo password or Duolingo revokes it server-side, which surfaces as a 401 and flips the connection to `TokenRevoked`.
- **Reconnect flow.** On `TokenRevoked`, sync pauses and the user is asked to paste a fresh token. This is the only maintenance a connected user ever performs, and only after a password change.
- **Encryption:** AES-256-GCM with a master key held in Azure Key Vault, read at startup via managed identity. Ciphertext blob = `nonce || tag || ciphertext`; `KeyVersion` recorded for rotation.
- **What the user is told, in plain words:** DuoVelocity's operator can technically access the stored token, because the sync process has to decrypt it to use it. The token lets the operator act as you *on Duolingo only*. Change your Duolingo password at any time to invalidate everything DuoVelocity holds.
- **Disconnect** purges the token ciphertext immediately; **delete my account** purges all rows for the user.

## 11. API (v1)

All routes under `/api/v1`, bearer auth required unless noted. JSON only. Dates in ISO-8601; metric series are arrays of `{ "date": "2026-09-14", "count": 3 }`.

| Method | Route | Purpose |
|---|---|---|
| GET | `/me` | profile, timezone, connection status summary |
| POST | `/duolingo/connect` | `{ jwt }` → validates the token with one authenticated call, stores it encrypted |
| DELETE | `/duolingo/connect` | disconnect; `?purgeData=true` also deletes history |
| GET | `/duolingo/status` | status, last sync, last success, token expiry, `gapDetected` |
| POST | `/duolingo/sync` | manual sync (enqueue); rate-limited to one per 3 min per user |
| GET | `/metrics/lessons-per-day?from&to` | lessons and activities as two series |
| GET | `/metrics/units-per-week?from&to` | |
| GET | `/metrics/units-per-month?from&to` | |
| GET | `/metrics/score-history?from&to` | |
| GET | `/metrics/summary` | rolling averages, last unit, last sync |
| GET | `/health` | anonymous liveness for the platform |

CORS: allow-list the frontend origin(s) once known. Errors: RFC 7807 problem details. OpenAPI document served in Development only.

## 12. Hosting, scheduling, cost

| Component | Azure resource | Free-tier facts that matter |
|---|---|---|
| Nightly sync | Function App, **Windows** Consumption, .NET 10 isolated | Always-free grant (1M executions/mo). Windows is required for `WEBSITE_TIME_ZONE`; on Linux Consumption the time-zone settings are unsupported and timers run in UTC. Storage account (queues) comes with it. |
| API | App Service **F1** | 60 CPU-minutes/day, 1 GB, `*.azurewebsites.net` only (no custom domain/TLS binding), no Always On — sleeps after ~20 min idle, cold start on first request. Exceeding CPU/bandwidth/filesystem quota stops the app (HTTP 403) until reset. No SLA. Upgrade path: Basic B1 (~$13/mo) for custom domain + Always On. |
| Database | Azure SQL Database, free offer (serverless) | 100k vCore-seconds + 32 GB/month, renews monthly. Auto-pauses when idle → first query after pause is slow (the 04:30 run and the day's first API call will both wake it). Set "when free amount is used: pause until next month" to guarantee $0. |
| Secrets | Key Vault (Standard) | Fractions of a cent per month at this volume |
| Telemetry | Application Insights | Free monthly ingestion allowance covers this |

Subscription: David's **personal** Azure subscription — nothing shared with client resource groups. Everything in one resource group, one region.

Timezones in .NET: `TimeZoneInfo.FindSystemTimeZoneById` accepts IANA ids cross-platform on .NET 6+, so the Duolingo `timezone` value can be used directly for `EventLocalDate`.

## 13. Observability
- Structured logging through `ILogger<T>` in `Core`; hosts wire Application Insights.
- `SyncRun` is the audit trail a user can see (`/duolingo/status`) and the operator can query.
- Alerts (App Insights log alert, email): any user at `ConsecutiveFailures >= 2`; any `GapDetected`; any unknown `eventType` or `NodeType` string encountered (logged at Warning with the raw value).
- CLI `inspect` command dumps the parsed view of a `RawSnapshot` for debugging without touching Azure.

## 14. Risks and open questions

| # | Risk / unknown | Mitigation / how it gets answered |
|---|---|---|
| 1 | **Unofficial API** — endpoints, auth flow, or field names can change without notice | Raw landing; parsers tolerant of extra/missing fields; alerts on unknown values; accept that a breaking change means a maintenance release. **M0 confirmed login is captcha-gated** (reCAPTCHA Enterprise token in the login body), so server-side login is off the table — token-only connect (§10.3) |
| 2 | Duolingo ToS — this access is not sanctioned | Personal-scale use; a few polite requests per user per night; document the risk to users at connect time |
| 3 | ~~JWT lifetime unknown~~ **Resolved (M0):** the token's `exp` is ~200 years out with `iat` unset, so it does not self-expire. Only a password change or server-side revocation kills it (surfaces as 401 → `TokenRevoked`). 14-day empirical revocation test running as of 2026-09-18 | Reconnect flow handles the revocation case; no re-login needed |
| 4 | ~~Score field location unknown~~ **Resolved (M0):** the Score is `currentCourse.scoreMetadata.reachedScore`; the course cap is `pathEndingScore` (§7.2). Score stays a v1 metric | — |
| 5 | `eventType` / `NodeType` full enumerations undocumented | Log unknowns; build the list from real data. M0 saw `eventType` ∈ {`LESSON`, `PRACTICE`, `null`}; node `type` ∈ {`skill`, `story`, `practice`, `duo_radio`, `chest`, `unit_review`}; `state`/`nodeState` includes `passed`; `touchPointType` ∈ {`NORMAL`, `UNIT_END`} (§7.1, §9.3). Treat "not `LESSON`" as activity rather than matching a fixed activity type |
| 6 | Lesson-to-node ratio may be 1 or N | Never assumed; both counted independently |
| 7 | F1 quotas (CPU/day) could stop the API under real traffic | Monitor quotas; B1 upgrade is the escape hatch |
| 8 | SQL free tier exhaustion at multi-user scale | "Pause until next month" setting; watch vCore-seconds; the nightly fan-out is bursty, so consider spreading users across the 04:00–05:00 hour if it becomes a problem |
| 9 | Multi-user obligations: privacy policy, data deletion, credential disclosure | Disconnect/delete endpoints in v1; short plain-English privacy page before public launch |
| 10 | Consumption-plan cold start at 04:30 | Irrelevant to correctness; sync is not latency-sensitive |

## 15. Build order

| Milestone | Deliverable | Answers |
|---|---|---|
| **M0 — Spike** (CLI only, no DB) | ~~`login`~~ token capture from browser, `decode-token`, `dump` (raw JSON to disk, redacted fixture for `samples/`) | **Done:** login is captcha-gated (token-only connect); JWT does not self-expire; users endpoint `2017-06-30/users/{id}` works with a Bearer token. Score field resolved (`scoreMetadata.reachedScore`, §7.2). Still open: full `NodeType`/`eventType` enumerations, `totalSessions` ratio on more data. 14-day revocation test running |
| **M1 — Core + CLI** | Parsers, `PathDiffer` with tests on fixture JSON, `XpEvent` ingest, metrics queries; local SQL (LocalDB) | Diff algorithm correct on real data (David's own history) |
| **M2 — Azure nightly** | Free SQL DB, Function App (timer + queue), Key Vault, App Insights; David as the only connected user | Runs unattended for two weeks |
| **M3 — API + identity** | Entra External ID, connect/disconnect flow, metrics endpoints, F1 deploy, privacy page | Second user (family) connects successfully |
| **M4 — Frontend** | TBD | — |

## Appendix A — Decisions log

| Date | Decision |
|---|---|
| 2026-09-15 | All C#: Core library + CLI + Functions + ASP.NET Core Web API (Python dropped) |
| 2026-09-15 | Multi-user from day one |
| 2026-09-15 | Nightly sync 04:30 US Eastern; Windows Consumption for time-zone support |
| 2026-09-15 | API on App Service F1; frontend TBD |
| 2026-09-15 | ~~Store JWT always, password opt-in; re-login on expiry only if password stored~~ |
| 2026-09-18 | **Superseded by M0 findings:** login is captcha-gated, so no server-side login. Token-only connect; no password ever stored; reconnect flow on 401 (`TokenRevoked`). JWT confirmed non-expiring (`exp` ~200 yrs, `iat` unset) |
| 2026-09-18 | M0 timestamp audit (§7.1): path has no completion dates (confirmed); `xpGains[].time` is the only per-lesson timestamp and ages out in ~1–2 weeks (14 days observed); `eventType` can be `null`; other timestamps (creationDate, streakData) are account/streak-level, day-resolution, not a completion source |
| 2026-09-18 | M0 Score found (§7.2): Score is `currentCourse.scoreMetadata.reachedScore`, range 0–130 with a per-course cap (`pathEndingScore`), CEFR-aligned. Corrects the earlier "0–160". Score and XP are distinct metrics |
| 2026-09-18 | Primary objective stated as exactly three tracked things (§1): `LESSON` events, Units, Score. Practice/activity, XP totals, streak, leagues are explicitly not tracked |
| 2026-09-18 | Added a fourth tracked thing: **Daily XP** (`xp_summaries`), as a consistency signal. Cheap and ~15-month backfillable; stored in `DailyXp`. Still not tracked: lifetime XP-total dashboards, streak, leagues |
| 2026-09-21 | **Correction:** the path is ~9 MB parsed but only ~575 KB gzipped on the wire (the web app fetches it the same way, not cached). Earlier claim that the app avoids the full pull was a measurement error. Store `RawSnapshot` gzip-compressed (~200 MB/user/year, well inside 32 GB), not decompressed JSON (~3 GB/user/year) |
| 2026-09-15 | No-login/public-profile mode rejected — lessons and units require auth |
| 2026-09-15 | Vocabulary fixed: Section, Unit, Node, Lesson, Activity, Score; "level" banned |
| 2026-09-15 | Land raw JSON for every pull; store node *transitions*, not nightly node state |
| 2026-09-15 | Current course only in v1 |
| 2026-09-15 | v1 velocity metrics: lessons/day, units/week, units/month, Score. Node metrics dropped; nodes still tracked as supporting data |
| 2026-09-15 | Manual sync rate limit: one per 3 minutes per user |
