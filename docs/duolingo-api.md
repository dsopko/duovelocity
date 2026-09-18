# Duolingo Unofficial API — Reference

**Status:** working notes · last updated 2026-09-18
**Scope:** the subset of Duolingo's unofficial API that DuoVelocity depends on.

Duolingo publishes no official API. This document records the endpoints the web client uses, as far as DuoVelocity needs them. Two sources feed it, marked throughout:

- **[M0]** — verified live against a real account on 2026-09-18 (see `../samples/`).
- **[legacy]** — from the community project `bartsonb/duolingo-activity-history` (an older endpoint we did **not** re-test in this session; it may or may not still work).
- **[community]** — widely documented behavior, not independently verified here.

Everything here can change without notice. Treat field names as observed, not contracted.

---

## 1. Base URL and versioning

Base host: `https://www.duolingo.com`

Duolingo versions endpoints by **calendar date** embedded in the path (like Stripe's dated API versions). The date is a frozen version tag, not a timestamp of the data. Versioning is **per endpoint** — different endpoints carry different dates and do not move together:

| Endpoint family | Version segment | Source |
|---|---|---|
| User read | `/2017-06-30/users/{id}` (also `/2023-05-23/users/{id}`, used by the web app) | [M0] |
| Daily activity | `/2023-05-23/users/{id}/xp_summaries` | [M0] |
| Login | `/2023-05-23/login` | [M0] |
| Legacy user read | `/users/{username}` (no version) | [legacy] |

If a version is deprecated, the fix is to bump the date segment and re-check field names, not to change the rest of the call.

---

## 2. Authentication

**Credential: a JWT.** Obtained by logging in; thereafter it is the only credential needed.

- **Transport:** send it either as `Authorization: Bearer <jwt>` [M0] or as the `jwt_token` cookie [legacy — the bartsonb scraper sends the cookie value in a `Cookie` header]. In a browser it rides automatically as an `httpOnly` cookie named `jwt_token`, so page JavaScript cannot read it; DevTools → Application → Cookies can.
- **Claims [M0]:** minimal — `{ sub, iat, exp }`. `sub` is the numeric user id. Observed `iat: 0` and `exp: 6307200000` (~year 2170), i.e. **the token does not self-expire**. It dies only on a Duolingo password change or server-side revocation, which surfaces as HTTP 401.
- **Getting a token headlessly is not practical:** login is captcha-gated (§5). The realistic flow is: user signs in via browser, the token is captured from the session.

---

## 3. `GET /2017-06-30/users/{id}` — primary user read [M0]

The main data source. Requires an explicit `fields` list; the response contains only the requested fields.

**Request**
```
GET https://www.duolingo.com/2017-06-30/users/{id}?fields=xpGains,currentCourse
Authorization: Bearer <jwt>          # or send the jwt_token cookie
Accept: application/json
```

- `{id}` = the `sub` from the JWT (e.g. `27437326`).
- `fields` = comma-separated top-level fields. Nested objects come back whole.

**Fields DuoVelocity uses**

| Field | Contents |
|---|---|
| `id`, `username`, `timezone`, `creationDate` | identity, day bucketing, account age (`creationDate` is epoch seconds) |
| `courses[]` | course registry: `id`, `title`, `learningLanguage`, `fromLanguage`, `xp`, `crowns` |
| `xpGains[]` | recent lesson/activity events — see §6. **~7-day retention.** |
| `currentCourse` | the current course incl. the full path and the Score — see §7 |
| `streakData` | streak length and boundary dates (day resolution; not per-lesson) |

**Size note [M0]:** requesting `currentCourse` returns the entire path in one response — ~9 MB / 7,635 nodes / 992 units for a mid-course Spanish learner. Land it verbatim, then persist only transitions.

**Scope caveats [M0/community]:** returns the **current course only** (the one tied to the last-practiced language), and `xpGains` is only the last ~7 days.

---

## 4. `GET /users/{username}` — legacy user read [legacy]

The older endpoint used by `bartsonb/duolingo-activity-history`. Keyed by **username**, unversioned. Documented here for completeness and as a fallback event source; not re-verified in this session.

**Request**
```
GET https://www.duolingo.com/users/{username}
Cookie: <jwt_token value>
```

**Response shape (relevant part)**
```
{
  "language_data": {
    "<lang>": {
      "calendar": [
        { "skill_id": "...", "datetime": <ms?>, "improvement": <xp> },
        ...
      ]
    }
  }
}
```

- `language_data` is keyed by language code; each has a `calendar[]` activity feed.
- **Limitations (from its README):** only the **last ~8 days**, and only the language of the **last completed lesson**.

This `calendar` is the legacy equivalent of `xpGains` (§6). Field mapping:

| Legacy `calendar` | Modern `xpGains` | Meaning |
|---|---|---|
| `skill_id` | `skillId` | the skill the event belongs to |
| `datetime` | `time` | event timestamp (legacy appears to be ms; modern is epoch **seconds**) |
| `improvement` | `xp` | XP earned by the event |

---

## 5. `POST /2023-05-23/login` — login [M0]

**Captcha-gated.** The request body carries a reCAPTCHA Enterprise token that only a real browser can mint, so a headless server cannot complete this call. It is documented for completeness; do not build server-side login on it.

**Request**
```
POST https://www.duolingo.com/2023-05-23/login?fields=
Content-Type: application/json

{
  "identifier": "<username or email>",
  "password": "<password>",
  "signal": { "siteKey": "<recaptcha site key>", "token": "<recaptcha token>", "vendor": "..." },
  "distinctId": "<uuid>",
  "landingUrl": "...",
  "lastReferrer": "..."
}
```

**Response**
- On success, the JWT is returned in a `jwt` **response header** (and the `jwt_token` cookie is set).
- Bad credentials or a missing/invalid captcha token → **HTTP 401**.

---

## 6. Activity events — `xpGains[]` [M0]

Every entry is one XP-earning event. Four fields:

```json
{ "xp": 40, "skillId": "75157f2f1304e0f4e064bb0e58e57fc5", "eventType": "LESSON", "time": 1788467441 }
```

| Field | Type | Notes |
|---|---|---|
| `time` | int (epoch **seconds**) | the **only per-lesson timestamp** in the whole payload; ages out in ~7 days |
| `eventType` | string or null | observed: `LESSON`, `PRACTICE`, `null` |
| `skillId` | string or null | present on `LESSON` events; `null` on `PRACTICE`/`null` events |
| `xp` | int | XP earned |

**Only `LESSON` events advance the path.** They carry a `skillId` that resolves to a path node → unit. `PRACTICE` and `null` events carry no `skillId` and are off-path activity. Bucket "not `LESSON`" as activity rather than matching a specific type.

---

## 7. `currentCourse` — path and Score [M0]

Top-level keys observed include: `title`, `learningLanguage`, `fromLanguage`, `xp`, `crowns`, `scoreMetadata`, `pathSectioned`, `sections`, `path`, `wordsLearned`, `status`, and more.

### 7.1 Score — `currentCourse.scoreMetadata`

The Duolingo Score (the CEFR-aligned number shown top-left on the path; users often call it their "level").

```json
{ "supportType": "FULLY_CEFR_ALIGNED", "reachedScore": 66, "pathStartingScore": 5, "pathEndingScore": 129 }
```

| Field | Meaning |
|---|---|
| `reachedScore` | current Score — the value to record over time |
| `pathStartingScore` / `pathEndingScore` | this course's Score range; **the cap is per-course — read it, don't hardcode** |
| `supportType` | e.g. `FULLY_CEFR_ALIGNED` |

Range is 0–130 on fully built courses (English, Spanish, French); lower elsewhere. Score rises with lessons, units, sections, **and** practice/stories — it is not a step function of unit completion.

### 7.2 Path structure — `pathSectioned[]`

```
currentCourse.pathSectioned[]        // sections
  └─ units[]                         // units
       └─ levels[]                   // nodes
```

**Unit object** (`pathSectioned[].units[]`):

| Field | Meaning |
|---|---|
| `unitIndex` | unit number (e.g. 194) |
| `levels[]` | the nodes in this unit |
| `teachingObjective` | e.g. "Destinations: Ask about travel plans" |
| `cefrLevel` | e.g. "B1" |
| `learningUnitType` | e.g. `INTERMEDIATE_IMMERSIVE_MINI_UNIT` |
| `guidebook.url` | link to the unit guidebook JSON |
| `referenceId`, `isUnlocked`, `isInIntro` | misc |

**Node object** (`units[].levels[]`):

| Field | Meaning |
|---|---|
| `type` | node kind — observed: `skill`, `story`, `practice`, `duo_radio`, `chest`, `unit_review` |
| `subtype` | e.g. `regular`, `read`, `chest`, `unit_review` |
| `state` | completion state — observed: `passed`, `active`, `locked`; treat `passed`/`legendary` as complete |
| `finishedSessions` / `totalSessions` | session counts; complete when equal (but see caveat below) |
| `pathLevelMetadata` | `nodeState`, `unitIndex`, and `skillId`/`anchorSkillId`/`storyId`/`duoRadioSummary` depending on type |
| `pathLevelClientData` | client hints incl. `skillId`(s), `cefr`, `teachingObjective` |
| `levelScoreInfo` | `reachedScore`, `learningScore`, **`touchPointType`**, `reachedProgress`, `completedProgress` |
| `absoluteNodeIndex` | global order of the node in the path |
| `debugName` | human label, e.g. "Unit Review 194" |

**No node carries a completion date.** Completion time must be derived (see §8).

### 7.3 Detecting a unit boundary

Two reliable structural signals [M0]:

- The **last node of a unit is `type: "unit_review"`** with `levelScoreInfo.touchPointType == "UNIT_END"`. Every other node reads `touchPointType: "NORMAL"`.
- Nodes self-identify their unit via `pathLevelMetadata.unitIndex`, and order via `absoluteNodeIndex`.

**Caveat:** the `unit_review` node can read `state: "passed"` with `finishedSessions: 0`, so rely on **`state`**, not the session counts, for that node type.

---

## 8. Timestamps and derivation

| Source | Resolution | Retention |
|---|---|---|
| `xpGains[].time` | second (epoch) | ~7 days |
| `xp_summaries[].date` (§11.1) | day (epoch) | ~15 months |
| `creationDate`, `streakData.*Timestamp` | second (epoch) | permanent (account/streak level) |
| `streakData.*Streak.*Date` | day (`YYYY-MM-DD`) | permanent (streak boundaries) |
| path nodes/units | **none** | — |

Because the path has no dates, node and unit completion times are **manufactured**:

1. When a snapshot shows a node newly complete, find `LESSON` events whose `skillId` matches the node and whose `time` is after the last sync. Use the latest such `time`. (`Attribution = LessonEvent`.)
2. If none remain in the window, fall back to the snapshot day. (`Attribution = SnapshotDay`.)
3. A unit's completion time = the latest node completion among its progression nodes (`unit_review` UNIT_END being the terminal one).

A skill id can appear on multiple nodes in a unit (crown levels), so `skillId` narrows a lesson to a skill and unit, not always one exact node.

---

## 9. Enumerations observed [M0]

- **`eventType`:** `LESSON`, `PRACTICE`, `null`
- **node `type`:** `skill`, `story`, `practice`, `duo_radio`, `chest`, `unit_review`
- **node `state` / `nodeState`:** `passed`, `active`, `locked` (also `legendary` per design)
- **`touchPointType`:** `NORMAL`, `UNIT_END`
- **`learningUnitType`:** `INTERMEDIATE_IMMERSIVE_MINI_UNIT` (others likely exist)
- **`supportType`:** `FULLY_CEFR_ALIGNED`

These lists are from one account's current course; log unknown values and extend the lists from real data.

---

## 10. Known limitations and quirks

- **~7–8 day retention applies only to per-lesson data.** `xpGains` (and legacy `calendar`) hold about a week, so a missed week is an irrecoverable gap for *lesson-level* detail (skill ids, lesson/practice split). **Daily totals do not have this limit:** `xp_summaries` (§11.1) exposes ~15 months of per-day XP and session counts, backfillable at connect time. Node/unit completions are still detected by path diff, but lose precise timestamps when the finishing lessons age out.
- **Current course only.** The user read returns the last-practiced course; multi-course tracking needs separate handling.
- **No path timestamps.** Completion dates are always derived, never read.
- **Captcha-gated login.** No headless password login; the token is the durable credential.
- **Per-course Score cap.** Read `pathEndingScore`; do not assume 130 or 160.
- **Large payloads.** `currentCourse` is multi-MB; land raw, persist transitions.

---

## 11. Lightweight and targeted endpoints [M0]

Found by watching the web app load `/learn` (2026-09-18): ~54 requests, nearly all under 5 KB. **The app never pulls the full course path on load** — the 9 MB `currentCourse` response is a consequence of DuoVelocity asking for that whole field, not an app default. The app's heaviest call is ~575 KB (`/2023-05-23/users/{id}` with a large profile/settings/courses `fields` list, which does *not* include the full path). These small endpoints cover most of DuoVelocity's needs cheaply.

### 11.1 `GET /2023-05-23/users/{id}/xp_summaries` — daily activity (recommended)

A compact per-day rollup, and the important find: **retention is ~15 months**, not 7 days. Observed 462 daily entries (2025-05-06 → 2026-09-18) in ~83 KB. Query with `startDate`.

```
GET https://www.duolingo.com/2023-05-23/users/{id}/xp_summaries?startDate=2025-01-01
Authorization: Bearer <jwt>
```

Response: `{ "summaries": [ ... ] }`. Each entry:

| Field | Meaning |
|---|---|
| `date` | epoch seconds, local midnight of the day |
| `gainedXp` | total XP earned that day |
| `numSessions` | number of sessions completed that day |
| `totalSessionTime` | seconds spent in sessions |
| `streakExtended`, `frozen`, `shielded`, `repaired` | streak bookkeeping |
| `dailyGoalXp` | that day's XP goal |

**Use and limits:** the cheap, long-history source for *daily activity*; it enables **~15-month backfill at connect time** instead of only observing forward. But it is a daily rollup — `numSessions` counts all sessions (lessons **and** activities) with no `skillId` and no lesson/practice split. It does **not** replace `xpGains` (§6) for per-lesson attribution, nor the path diff for unit completions.

### 11.2 `GET /2023-05-23/users/{id}` — newer user read

The version the web app itself uses for the main user object (vs. our `2017-06-30`). Same `fields`-list mechanism. Either version serves the fields DuoVelocity needs; `2017-06-30` is the one verified for `xpGains` and `currentCourse`.

### 11.3 `GET /2023-05-23/score-info/courses/{COURSE}?fields=scores` — Score (observed, shape unconfirmed)

Seen in app traffic returning ~0.4 KB with the Score (e.g. `COURSE = DUOLINGO_ES_EN`). A direct replay returned HTTP 400, so the app sends request context not yet reproduced. Until confirmed, read the Score from `currentCourse.scoreMetadata` (§7.1).

### 11.4 Other small endpoints seen on load

All sub-2 KB, not needed by DuoVelocity, noted for orientation: `/users/{id}/streak-goal-current`, `/streak-goal-next-options`, `/2017-06-30/users/{id}/courses/{learning}/{from}/learned-lexemes/count`, `/practice-lexemes`, `/2023-05-23/shop-items`, `/2017-06-30/messaging/get-messages/`, `/2017-06-30/friends/...`, `/quests`.

### 11.5 Sourcing DuoVelocity's data cheaply

| Need | Cheap source | Full path (9 MB) required? |
|---|---|---|
| Daily activity (sessions/day, XP/day), backfillable ~15 mo | `xp_summaries` (§11.1) | No |
| Lessons vs activities split (last ~7 days) | `xpGains` (§6) | No |
| Score | `currentCourse.scoreMetadata` (§7.1) | No (small `fields` pull) |
| Unit completions (units/week, units/month) | path diff (§7.2, §8) | Yes, but only on nights with new lessons |

## 12. References

- `bartsonb/duolingo-activity-history` — legacy `/users/{username}` + `calendar` [legacy]
- `duoplanet.com/duolingo-score` — Score/CEFR bands [community]
- `../samples/` — live captures used to verify the [M0] items:
  - `es_en-unit194-raw.json` (one completed unit, all fields)
  - `duolingo-full-es_en.json` (entire user read; not committed — personal data)
