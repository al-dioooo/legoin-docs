# Legoin API Contract

- Status: Draft, under review in #7
- Version: 0.2
- Date: 2026-11-25
- Issue: #7
- Source of the shapes: `src/lib/domain.ts` and `src/lib/api` in [legoin-web](https://github.com/al-dioooo/legoin-web), on `main` at `a05024d`

This is the contract between legoin-web and the Python backend. The backend runs the models (detection, then part classification and color, as in [legoin-ml](https://github.com/al-dioooo/legoin-ml)) and an LLM agent that suggests builds. Both sides build against this document, independently.

In legoin-web, screens read and write only through the data layer, `src/lib/api`. Today a browser-side mock serves it from `localStorage`. Setting `NEXT_PUBLIC_API_URL` will switch it to an HTTP client for this API. Every function in the data layer has an endpoint here, and every shape matches `domain.ts`, so the switch changes nothing outside `src/lib/api`.

Parts the team hasn't decided yet are marked **Open question** and collected in [Open questions](#10-open-questions). Technical choices this draft makes on its own are collected in [Decisions in this draft](#11-decisions-in-this-draft), so a reviewer can push back on them.

## 1. Overview

| Data layer (`src/lib/api`) | Endpoint | Answer |
| --- | --- | --- |
| `register(email, password)` | [`POST /auth/register`](#post-authregister) | A `Session`, which the data layer keeps. It resolves with the `Account`. |
| `logIn(email, password)` | [`POST /auth/login`](#post-authlogin) | The same |
| `logOut()` | [`POST /auth/logout`](#post-authlogout) | 204. The data layer forgets the session either way. |
| `getAccount()` | [`GET /auth/me`](#get-authme) | `Account`, or 401, which the client turns into `null` |
| `createScan(photo, { onStage, signal })` | [`POST /scan-jobs`](#post-scan-jobs), then [`GET /scan-jobs/{jobId}`](#get-scan-jobsjobid) until it ends. [`DELETE /scan-jobs/{jobId}`](#delete-scan-jobsjobid) cancels. | A `ScanJob` that ends with a `Scan` |
| `getScan(id)` | [`GET /scans/{scanId}`](#get-scansscanid) | `Scan`, or 404, which the client turns into `null` |
| `listScans()` | [`GET /scans`](#get-scans) | `{ scans: Scan[] }`, newest first |
| `updateDetections(id, changes)` | [`PATCH /scans/{scanId}/detections`](#patch-scansscaniddetections) | The updated `Scan` |
| `deleteScan(id)` | [`DELETE /scans/{scanId}`](#delete-scansscanid) | 204 |
| `getIdeas(scanId, theme, locale, { exclude })` | [`POST /scans/{scanId}/ideas`](#post-scansscanidideas) | An `IdeaSet` of 3 ideas |
| `getIdea(scanId, ideaId)` | [`GET /scans/{scanId}/ideas/{ideaId}`](#get-scansscanidideasideaid) | `StoredIdea`, or 404, which the client turns into `null` |

A visit goes: log in or make an account, upload a photo and follow the scan's stages, check and correct the pieces, ask for ideas in a theme, open one idea, and come back to the scan later from history, on any device.

## 2. Conventions

- **Base URL.** `NEXT_PUBLIC_API_URL` is the base, version included, e.g. `https://api.legoin.example/v1`. Paths below are relative to it. A breaking change gets a new version (`/v2`).
- **JSON.** Requests and answers are JSON in UTF-8, except the photo upload, which is `multipart/form-data`. Keys are camelCase, with the same names as in `domain.ts`.
- **Optional fields** are left out when they have no value, never sent as `null`. The exceptions are a detection's `box` and `confidence`, which are always present and `null` for a piece added by hand.
- **Unknown fields.** A client ignores fields it doesn't know, so the backend can add an optional field within a version.
- **Times** are ISO 8601 in UTC, ending in `Z`, e.g. `2026-11-09T03:12:45.120Z`. Sorting them as strings sorts them by time.
- **Ids** are opaque strings the backend makes: URL-safe (`A–Z a–z 0–9 _ -`), at most 64 characters. The client never parses or makes one; a `restore` change sends a detection back with the id the backend gave it.
- **Whose scans.** Each scan belongs to the account that made it, and an account only ever sees its own. Every request carries the account's session as `Authorization: Bearer <token>`, except the requests that make a session and the URLs an `<img>` loads. See [Accounts](#4-accounts).
- **CORS.** The data layer runs in the browser, in Client Components, so the browser calls the API directly. The API allows the web app's origins, the methods `GET`, `POST`, `PATCH` and `DELETE`, and the request headers `Content-Type` and `Authorization`. The API sets no cookies, so it never sends `Access-Control-Allow-Credentials`.
- **Errors** share one body. See [Errors](#7-errors).
- **Examples** use `https://api.legoin.example/v1` as the base. The ones marked `jsonc` are shortened with comments; real answers are plain, complete JSON.

## 3. Shapes

Written as TypeScript because `domain.ts` is; the backend can mirror them with Pydantic models. Everything here is in `domain.ts` or `bricks.ts` with the same names and fields, except `ScanJob`, `Session` and `ApiErrorBody`, which only exist on the wire. `Account` and `Session` came with accounts ([ADR-0011](../decisions/0011-accounts-with-email-and-password.md)). `Idea`'s `buildId` and `preview`, and `Preview`, were added after `a05024d` ([ADR-0010](../decisions/0010-library-of-checked-builds.md)).

```ts
type Scan = {
  id: string;
  createdAt: string; // ISO 8601, UTC: when the scan finished
  photo: Photo;
  detections: Detection[];
  lastTheme?: Theme; // the theme last used for ideas
  ideas?: IdeaSet; // the latest ideas; left out when there are none
};

type Photo = { src: string; width: number; height: number };

type Detection = {
  id: string;
  part: BrickPartId;
  color: BrickColorId;
  box: Box | null; // null for a piece added by hand
  confidence: number | null; // 0 to 1; null for a piece added by hand
  edited?: boolean; // true once the person changed or confirmed it
};

/** Fractions of the photo, 0 to 1, from its top-left corner. */
type Box = { x: number; y: number; width: number; height: number };

type DetectionChange =
  | { type: "edit"; id: string; part?: BrickPartId; color?: BrickColorId }
  | { type: "remove"; id: string }
  | { type: "add"; part: BrickPartId; color: BrickColorId; quantity: number }
  | { type: "restore"; detection: Detection };

/** One inventory or idea line: how many of one part in one color. */
type PartCount = { part: BrickPartId; color: BrickColorId; quantity: number };

type Idea = {
  id: string;
  theme: IdeaTheme; // never "surprise"
  difficulty: Difficulty;
  title: string;
  summary: string;
  steps: string[];
  used: PartCount[]; // parts the person owns that the build uses
  missing: PartCount[]; // parts the build needs that the person doesn't have
  buildId?: string; // the checked build this idea is, from the backend's library
  preview?: Preview; // pictures of that build; sent with buildId
};

/** A checked build's picture and LDraw model, as absolute URLs. */
type Preview = { image: string; model: string };

type IdeaSet = { theme: Theme; locale: Locale; items: Idea[] };

type StoredIdea = { idea: Idea; set: IdeaSet };

/** Who's logged in. */
type Account = {
  id: string;
  email: string; // trimmed and lowercased by the backend
  createdAt: string; // ISO 8601, UTC: when the account was made
};

type ScanStage = "detecting" | "classifying" | "coloring";
type Theme = "vehicles" | "animals" | "buildings" | "surprise";
type IdeaTheme = "vehicles" | "animals" | "buildings";
type Difficulty = "easy" | "medium" | "hard";
type Locale = "en" | "id";

// Only on the wire:

type ScanJob =
  | { id: string; status: "running"; stage: ScanStage }
  | { id: string; status: "done"; scan: Scan }
  | { id: string; status: "failed"; error: ApiErrorBody["error"] };

type Session = {
  token: string; // opaque; sent back as `Authorization: Bearer <token>`
  expiresAt: string; // ISO 8601, UTC
  account: Account;
};

type ApiErrorBody = {
  error: { code: ErrorCode; message: string; field?: string };
};
```

`ErrorCode` is listed in [Errors](#7-errors), and `BrickPartId` and `BrickColorId` in [the catalogue](#the-catalogue).

### Photo

- `src` is an absolute URL that a plain `<img>` can load, with no extra headers, for as long as the scan exists. It needs no `Authorization` header either: the API gives it only to the scan's account, and its name can't be guessed ([ADR-0011](../decisions/0011-accounts-with-email-and-password.md)). Where photos are stored, and for how long, is an **open question** ([3](#10-open-questions)).
- `width` and `height` are that image's size in pixels, upright. The backend applies the photo's EXIF orientation before it detects or stores anything, so boxes and pixels agree.

### Detection

- **Boxes** are fractions of the photo: `x` and `y` are the top-left corner, and `x + width` and `y + height` are at most 1. They scale with the image, whatever size it's shown at.
- **A piece added by hand** has `box: null` and `confidence: null`. Every piece the detector found has both.
- **Confidence** is one number from 0 to 1 for the whole reading, part and color together. How the backend combines the three stages' scores is its choice; the lowest of them is a safe default. The web app flags a piece below 0.6 (`LOW_CONFIDENCE`) for checking until it's `edited`.
- **`edited`** is `true` once any `edit` change has touched the piece, which includes confirming it with its own part and color, and for every piece added by hand. Otherwise it's left out; `false` means the same. The backend never sets it on its own.
- A changed piece keeps its box and its original confidence.

### Inventory

The inventory isn't sent. It's the scan's detections grouped by part and color, as `PartCount` lines: `toInventory` in legoin-web's `src/lib/inventory.ts`. Lines are sorted biggest quantity first, then by part id, then by color id. The backend uses the same grouping to check ideas and to tell whether a correction changed the inventory. The issue's `InventoryLine` is this `PartCount`.

### Themes and difficulty

| Theme | What the web app promises |
| --- | --- |
| `vehicles` | Cars, trucks and boats |
| `animals` | Ducks, turtles and penguins |
| `buildings` | Houses, lighthouses and bridges |
| `surprise` | One idea from each theme. It only appears in a request and in `IdeaSet.theme`; each idea in the set has its own theme, one of the three above. |

Difficulty is `easy`, `medium` or `hard`.

### Preview

Some ideas are a checked build from the backend's library: built from real parts, checked for collisions, floating parts and connectivity, and rendered ahead of time ([ADR-0010](../decisions/0010-library-of-checked-builds.md)). Every template idea is one, and the agent may pick one when the person's pieces fit it. Those ideas carry `buildId` and `preview`; an idea the agent wrote itself has neither.

- **`buildId`** is the build's id in the library, e.g. `race-car`. Like other ids, the client never parses it. The idea's title, summary, steps and parts are that build's, in the idea's language. `used` and `missing` still come from `cek_inventori`.
- **`preview.image`** is an absolute URL of a picture of the finished build: an isometric view, a square WebP of 640 × 640 pixels on a transparent background. It loads in a plain `<img>`, with no header.
- **`preview.model`** is an absolute URL of the build as an LDraw file (`.ldr`), served as a download. It names parts from the LDraw.org Parts Library and doesn't include them.
- **Both URLs are public and versioned.** They carry nothing about the account, and a URL never changes what it serves, so clients and proxies may cache it for good. Where the backend serves them is its choice.
- **`buildId` and `preview` come together:** an idea has both or neither.
- **The client** writes the picture's alt text itself, from the title, in the page's language. Wherever it shows pictures, it credits the LDraw.org Parts Library, BrickLayer and BrickWright, and the LEGO trademark line, as ADR-0010 lists.

### The catalogue

Part and color ids on the wire are the app's own ids from legoin-web's `src/lib/bricks.ts`, not Rebrickable ids. Issue #7 asked for Rebrickable color ids, but `domain.ts` and every screen use these ids, so the contract follows the code. Each color maps to one Rebrickable color id and each part to one LEGO design id, so the backend translates model output to these ids in one table.

The catalogue is the PRD's MVP: 10 colors and 13 parts. The backend only ever sends these ids. Adding a part or a color is a contract change, and legoin-web has to learn the new id before the backend sends it.

| Color id | English | Indonesian | Rebrickable id | Rebrickable name |
| --- | --- | --- | --- | --- |
| `red` | Red | Merah | 4 | Red |
| `blue` | Blue | Biru | 1 | Blue |
| `yellow` | Yellow | Kuning | 14 | Yellow |
| `green` | Green | Hijau | 2 | Green |
| `black` | Black | Hitam | 0 | Black |
| `white` | White | Putih | 15 | White |
| `light-gray` | Light gray | Abu-abu terang | 71 | Light Bluish Gray |
| `dark-gray` | Dark gray | Abu-abu gelap | 72 | Dark Bluish Gray |
| `orange` | Orange | Oranye | 25 | Orange |
| `brown` | Brown | Cokelat | 70 | Reddish Brown |

| Part id | Name | Design id |
| --- | --- | --- |
| `brick-1x1` | Brick 1×1 | 3005 |
| `brick-1x2` | Brick 1×2 | 3004 |
| `brick-1x3` | Brick 1×3 | 3622 |
| `brick-1x4` | Brick 1×4 | 3010 |
| `brick-2x2` | Brick 2×2 | 3003 |
| `brick-2x3` | Brick 2×3 | 3002 |
| `brick-2x4` | Brick 2×4 | 3001 |
| `plate-1x2` | Plate 1×2 | 3023 |
| `plate-2x2` | Plate 2×2 | 3022 |
| `plate-2x4` | Plate 2×4 | 3020 |
| `slope-2x2` | Slope 45° 2×2 | 3039 |
| `tile-1x2` | Tile 1×2 | 3069b |
| `round-1x1` | Round 1×1 | 3062b |

## 4. Accounts

Scanning needs an account ([ADR-0011](../decisions/0011-accounts-with-email-and-password.md)). A person makes one with an email and a password, or logs in to one, and the API answers with a `Session`. The data layer keeps it and sends its token on every other request:

```http
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

Each account sees only its own scans, from any device it logs in on.

| Needs a session | Public |
| --- | --- |
| Every route in [Scans](#5-scans) and [Ideas](#6-ideas), and `GET /auth/me` | `POST /auth/register`, `POST /auth/login` and `POST /auth/logout`, and the URLs an `<img>` loads: `photo.src` and `preview` |

### Sessions

- **The token** is opaque: the client never parses it and sends it back as it came. The API's tokens are `lgn_` and 43 URL-safe characters.
- **It lasts until `expiresAt`,** 30 days after logging in. There's no refresh; after that, the person logs in again.
- **Each log-in is a session of its own,** so an account can be logged in on several devices at once. Logging out ends only the session it's sent with.
- **Without a valid session,** a route that needs one answers `401 unauthorized`, with `WWW-Authenticate: Bearer`, before any other check. That's no `Authorization` header, a header that isn't `Bearer <token>`, or a token that's unknown, expired or logged out.
- **Another account's scan, job or idea** answers `404 not_found`, the same as one that doesn't exist. The API never answers `403`, which would tell the caller that the id exists.
- **The client** keeps the session in `localStorage` and sends the token only to `NEXT_PUBLIC_API_URL`. On `401 unauthorized`, it forgets the session and sends the person to log in, then back to the page they were on.

### Email and password

| Field | Rules |
| --- | --- |
| `email` | At most 254 characters, `name@domain` with a dot in the domain. The backend trims and lowercases it and keeps it that way, so `Ana@Example.com ` and `ana@example.com` are one account. Nothing is sent to it, so it's never confirmed. |
| `password` | 8 to 128 characters, any characters, with no other rules. The backend keeps only an Argon2id hash and never logs it. |

- **A field that breaks a rule** answers `400 invalid_request`, with `email`, `password` or `deviceId` in `field`.
- **Logging in** checks only that the email and password are there and not too long, so a password made under older rules still works.
- **Too many failures.** After 10 failed log-ins for one email within 15 minutes, `POST /auth/login` for that email answers `429 rate_limited`, with `Retry-After` in seconds, until the oldest of those failures is 15 minutes old. A log-in that works clears the count.

### `POST /auth/register`

`register(email, password)`: makes an account and logs it in. It answers `201` with a `Session`.

| Field | Type | Meaning |
| --- | --- | --- |
| `email` | `string` | See [Email and password](#email-and-password). |
| `password` | `string` | The same. |
| `deviceId` | `string`, optional | The `Legoin-Device-Id` this browser used before accounts, to move its scans. See [Scans from before accounts](#scans-from-before-accounts). |

```http
POST /v1/auth/register HTTP/1.1
Host: api.legoin.example
Content-Type: application/json

{ "email": "ana@example.com", "password": "baseplate-2x4", "deviceId": "3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10" }
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "token": "lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg",
  "expiresAt": "2026-12-25T03:10:00.000Z",
  "account": { "id": "acc_7r2k9d", "email": "ana@example.com", "createdAt": "2026-11-25T03:10:00.000Z" }
}
```

Errors: `400 invalid_request`, and `409 email_taken` when the email already has an account.

### `POST /auth/login`

`logIn(email, password)`: logs in to an account, as a new session, and answers `200` with a `Session` in the same shape. The body has the same fields as `POST /auth/register`.

Errors: `400 invalid_request`; `401 invalid_credentials` when no account has this email or the password is wrong, without saying which; `429 rate_limited`.

### `POST /auth/logout`

`logOut()`: ends the session whose token it carries, and answers `204` with no body. A missing, unknown or expired token answers `204` too, so logging out never fails. The data layer forgets the session either way.

```http
POST /v1/auth/logout HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```http
HTTP/1.1 204 No Content
```

### `GET /auth/me`

`getAccount()`: the session's `Account`. Without a valid session it's `401 unauthorized`, and the data layer returns `null`.

```http
GET /v1/auth/me HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```json
{ "id": "acc_7r2k9d", "email": "ana@example.com", "createdAt": "2026-11-25T03:10:00.000Z" }
```

### Scans from before accounts

Before accounts, the API kept scans by the `Legoin-Device-Id` header, a random id the browser made once. Those scans move to an account, once:

- `POST /auth/register` and `POST /auth/login` take that id as `deviceId`: 1 to 128 of `A–Z a–z 0–9 . _ -`, as the header was.
- When the request succeeds, every scan saved under that id that no account has yet moves to this account, with its photo and ideas. The answer is the same `Session`, and the scans show up in `GET /scans`.
- A scan that has moved never moves again, and a request that fails moves nothing. An id with no scans moves nothing and isn't an error.
- The web app sends the id it kept in `localStorage` (`legoin.deviceId.v1`), if it has one, then deletes it.

As before, the id is all it takes: whoever has it can move its scans, as they could read them before. `deviceId` goes once the deployed data has no scans left to move.

## 5. Scans

### Scanning a photo

`createScan(photo, { onStage, signal })` uploads a photo, reports the pipeline's stages as they happen, and resolves with the saved scan. Over HTTP that's a scan job the client polls:

1. `POST /scan-jobs` uploads the photo and answers `202` with a running job.
2. The client asks `GET /scan-jobs/{jobId}` every 500 ms and passes each job's `stage` to `onStage`. The data layer's `inOrder` already drops repeats and steps back. A stage that starts and ends between two polls is never seen, and that's fine.
3. When the job is `done`, `createScan` resolves with its `scan`. When it's `failed`, `createScan` rejects with an `ApiError` whose reason comes from the job's error (see [the mapping](#scanfailure-reasons)).

**Cancelling.** When `signal` aborts, the client stops polling and rejects with an `AbortError` straight away. It also sends `DELETE /scan-jobs/{jobId}` without waiting for the answer (`fetch` with `keepalive`). The backend saves nothing for a cancelled job. If the upload itself is aborted before the `202` arrives, the client has no job id to cancel, so a job the backend already accepted finishes and its scan shows up in history. This draft accepts that rare case.

**Giving up.** A job that's still running 30 seconds after the upload ends `failed` with `timeout`. The client gives up after 45 seconds with no ending, or after 3 polls in a row that get no answer: it sends the `DELETE` and rejects with the reason `network`. One failed poll on its own is just retried.

#### `POST /scan-jobs`

Uploads one photo as `multipart/form-data`, in a part named `photo`:

- JPEG, PNG or WebP;
- at most 10 MB (10 × 1024 × 1024 bytes, `MAX_PHOTO_BYTES` in legoin-web's `src/lib/photo.ts`).

The web app sends the JPEG copy it shows on screen, with its longest side at most 1200 px (`readPhoto`). The backend checks the type and size again, since a client check is only a courtesy.

```http
POST /v1/scan-jobs HTTP/1.1
Host: api.legoin.example
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
Content-Type: multipart/form-data; boundary=legoin

--legoin
Content-Disposition: form-data; name="photo"; filename="photo.jpg"
Content-Type: image/jpeg

<JPEG bytes>
--legoin--
```

```http
HTTP/1.1 202 Accepted
Content-Type: application/json

{ "id": "job_8c1f2e", "status": "running", "stage": "detecting" }
```

Errors: `400 invalid_request` when there's no `photo` part, `413 photo_too_large`, `415 photo_type`. Problems the models find in the photo (`no_pieces`, `too_dark`, `too_blurry`) don't answer the upload; they end the job.

#### `GET /scan-jobs/{jobId}`

The job as it is now. A running job reports its stage:

```http
GET /v1/scan-jobs/job_8c1f2e HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```json
{ "id": "job_8c1f2e", "status": "running", "stage": "classifying" }
```

A finished job carries the saved scan, which `GET /scans/{scanId}` and `GET /scans` now return too:

```json
{
  "id": "job_8c1f2e",
  "status": "done",
  "scan": {
    "id": "scn_4kd2p9",
    "createdAt": "2026-11-09T03:12:45.120Z",
    "photo": {
      "src": "https://photos.legoin.example/scn_4kd2p9.jpg",
      "width": 1200,
      "height": 800
    },
    "detections": [
      {
        "id": "det_a1",
        "part": "brick-2x4",
        "color": "red",
        "box": { "x": 0.075, "y": 0.165, "width": 0.278, "height": 0.245 },
        "confidence": 0.94
      },
      {
        "id": "det_a2",
        "part": "slope-2x2",
        "color": "green",
        "box": { "x": 0.405, "y": 0.54, "width": 0.18, "height": 0.235 },
        "confidence": 0.82
      },
      {
        "id": "det_a3",
        "part": "brick-2x2",
        "color": "dark-gray",
        "box": { "x": 0.727, "y": 0.54, "width": 0.172, "height": 0.262 },
        "confidence": 0.58
      }
    ]
  }
}
```

A failed job carries the error, and nothing was saved:

```json
{
  "id": "job_8c1f2e",
  "status": "failed",
  "error": { "code": "too_dark", "message": "The photo is too dark to find pieces." }
}
```

Rules:

- The stage only moves forward: `detecting`, `classifying`, `coloring`. A job that's waiting in a queue reports `detecting`.
- A job ends `done` or `failed` and never changes after that. The backend keeps an ended job for at least 10 minutes, then may answer `404`.
- The poll itself worked, so a failed job still answers `200`. Only an unknown job is an HTTP error: `404 not_found`.

#### `DELETE /scan-jobs/{jobId}`

Cancels the job and answers `204` with no body. The backend stops at the next stage and saves nothing. If the job had already finished, it deletes the scan the job saved, so a cancelled scan never shows up in history. An unknown or forgotten job also answers `204`, so the client can send it without checking first.

```http
DELETE /v1/scan-jobs/job_8c1f2e HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```http
HTTP/1.1 204 No Content
```

### `GET /scans/{scanId}`

`getScan(id)`: one scan, with its detections and, once ideas were asked for, its `lastTheme` and latest `ideas`. The answer is the `scan` from the finished job above, as it is now. An unknown scan, or one from another account, answers `404 not_found`, and the client returns `null`.

```http
GET /v1/scans/scn_4kd2p9 HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{ "error": { "code": "not_found", "message": "No scan scn_4kd2p9." } }
```

### `GET /scans`

`listScans()`: this account's scans, newest first by `createdAt`, as full `Scan` objects. They're wrapped in an object so paging can be added later without a new version; this version has no paging. No scans answers `{ "scans": [] }`.

```http
GET /v1/scans HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```jsonc
{
  "scans": [
    {
      "id": "scn_4kd2p9",
      "createdAt": "2026-11-09T03:12:45.120Z",
      "photo": { "src": "https://photos.legoin.example/scn_4kd2p9.jpg", "width": 1200, "height": 800 },
      "detections": [/* the three detections above */]
    },
    {
      "id": "scn_9tq1m3",
      "createdAt": "2026-11-08T11:20:00.000Z",
      "photo": { "src": "https://photos.legoin.example/scn_9tq1m3.jpg", "width": 900, "height": 1200 },
      "detections": [
        {
          "id": "det_k7",
          "part": "brick-2x4",
          "color": "blue",
          "box": { "x": 0.271, "y": 0.361, "width": 0.47, "height": 0.286 },
          "confidence": 0.96
        }
      ]
    }
  ]
}
```

### `PATCH /scans/{scanId}/detections`

`updateDetections(id, changes)`: applies the person's corrections and answers `200` with the updated scan. The body is `{ "changes": DetectionChange[] }`, with at least one change.

| Change | Fields | What it does |
| --- | --- | --- |
| `edit` | `id`, and `part` or `color` or both, or neither | Sets the part and color given, and marks the piece `edited`. Sending the piece's own part and color, or neither, confirms a flagged piece. |
| `remove` | `id` | Removes the piece. |
| `add` | `part`, `color`, `quantity` from 1 to 20 | Adds `quantity` pieces the detector missed, one detection each, with a new id, `box: null`, `confidence: null` and `edited: true`. |
| `restore` | `detection`, whole | Puts the piece back exactly as given, id, box and confidence included: it replaces the piece with that id, or is added if there's none. The web app uses it for undo. |

Rules:

- **In order, all or nothing.** Changes apply in the order sent. If one is invalid, such as an unknown part or color id or a quantity of 0, the answer is `400 invalid_request` with the change's path in `field`, and nothing changes.
- **Unknown ids are ignored.** An `edit` or `remove` for an id the scan doesn't have changes nothing, so a retried request or an undo is safe to send twice.
- **Ideas follow the inventory.** The backend compares the inventory before and after. If it changed, the scan's `ideas` are dropped, because they were checked against the pieces as they were, and the next request for ideas checks the corrected ones. If it's the same, as when a flagged piece is confirmed, the `ideas` stay. Either way `lastTheme` stays.

Say the scan from the job above already had ideas for `vehicles`. This request recolors its flagged piece and adds a blue plate the detector missed. The inventory changed, so the answer has no `ideas`, but `lastTheme` is still there:

```http
PATCH /v1/scans/scn_4kd2p9/detections HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
Content-Type: application/json

{
  "changes": [
    { "type": "edit", "id": "det_a3", "color": "black" },
    { "type": "add", "part": "plate-1x2", "color": "blue", "quantity": 1 }
  ]
}
```

```json
{
  "id": "scn_4kd2p9",
  "createdAt": "2026-11-09T03:12:45.120Z",
  "photo": { "src": "https://photos.legoin.example/scn_4kd2p9.jpg", "width": 1200, "height": 800 },
  "detections": [
    {
      "id": "det_a1",
      "part": "brick-2x4",
      "color": "red",
      "box": { "x": 0.075, "y": 0.165, "width": 0.278, "height": 0.245 },
      "confidence": 0.94
    },
    {
      "id": "det_a2",
      "part": "slope-2x2",
      "color": "green",
      "box": { "x": 0.405, "y": 0.54, "width": 0.18, "height": 0.235 },
      "confidence": 0.82
    },
    {
      "id": "det_a3",
      "part": "brick-2x2",
      "color": "black",
      "box": { "x": 0.727, "y": 0.54, "width": 0.172, "height": 0.262 },
      "confidence": 0.58,
      "edited": true
    },
    {
      "id": "det_b7",
      "part": "plate-1x2",
      "color": "blue",
      "box": null,
      "confidence": null,
      "edited": true
    }
  ],
  "lastTheme": "vehicles"
}
```

Errors: `400 invalid_request`, `404 not_found`.

### `DELETE /scans/{scanId}`

`deleteScan(id)`: deletes the scan with its photo and ideas, and answers `204` with no body. An unknown scan also answers `204`, so deleting twice, or deleting every scan at once from history, never fails on one that's already gone.

```http
DELETE /v1/scans/scn_4kd2p9 HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```http
HTTP/1.1 204 No Content
```

## 6. Ideas

### `POST /scans/{scanId}/ideas`

`getIdeas(scanId, theme, locale, { exclude })`: asks the agent for three new ideas and answers `200` with the `IdeaSet`. The data layer resolves with its `items`. Which LLM provider runs the agent is an **open question** ([1](#10-open-questions)); nothing here depends on it.

| Field | Type | Meaning |
| --- | --- | --- |
| `theme` | `Theme` | The theme asked for. |
| `locale` | `Locale` | The language to write the ideas in: the page's language. |
| `exclude` | `string[]`, optional | For "Try other ideas": the ids of ideas already shown for this scan, oldest first. At most 50. |

What the backend does:

- **Writes three ideas** in `locale`, each checked against the scan's inventory as it is now (see [Rules](#8-rules)). For `surprise`, it's one idea from each of `vehicles`, `animals` and `buildings`.
- **Adds a picture** to each idea that's a checked build from its library: `buildId` and `preview` (see [Preview](#preview)).
- **Keeps them.** The set becomes the scan's `ideas`, replacing the one before, and `theme` becomes its `lastTheme`.
- **Avoids excluded ideas.** The backend remembers every idea it has written for a scan, not only the latest set, so it can tell the agent what was already shown. Ids it doesn't know are ignored.
- **Asks again every time.** Every call is a wait and gives different ideas. To show the kept ones, read `ideas` from `GET /scans/{scanId}` instead.

A request that fails changes nothing: the scan keeps its earlier `ideas` and `lastTheme`. When two requests for one scan overlap, the one that finishes last is the set that's kept.

Idea ids are unique within a scan and never reused for a different idea, so an old link to an idea that has since been replaced answers `404` rather than showing another idea.

This request comes after the correction above, in Indonesian, with three ideas already shown. Each idea is checked against the scan's four pieces: a red brick 2×4, a green slope 2×2, a black brick 2×2 and a blue plate 1×2. The first idea is the library's race car, so it has a picture; the other two were written by the agent.

```http
POST /v1/scans/scn_4kd2p9/ideas HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
Content-Type: application/json

{ "theme": "vehicles", "locale": "id", "exclude": ["idea_m2x9", "idea_p0d4", "idea_t6w1"] }
```

```json
{
  "theme": "vehicles",
  "locale": "id",
  "items": [
    {
      "id": "idea_r4c8",
      "theme": "vehicles",
      "difficulty": "easy",
      "title": "Mobil balap kecil",
      "summary": "Badan merah, kap hijau, lampu kuning.",
      "steps": [
        "Pasang brick 2×4 merah di atas plate 2×4 hitam, mundur satu stud, supaya baris depan plate kelihatan.",
        "Taruh slope 2×2 hijau di depan brick merah sebagai kap mesin, dan brick 2×2 hitam di belakangnya sebagai kabin.",
        "Pasang dua round 1×1 kuning di baris depan plate sebagai lampu depan."
      ],
      "used": [
        { "part": "brick-2x2", "color": "black", "quantity": 1 },
        { "part": "brick-2x4", "color": "red", "quantity": 1 },
        { "part": "slope-2x2", "color": "green", "quantity": 1 }
      ],
      "missing": [
        { "part": "round-1x1", "color": "yellow", "quantity": 2 },
        { "part": "plate-2x4", "color": "black", "quantity": 1 }
      ],
      "buildId": "race-car",
      "preview": {
        "image": "https://api.legoin.example/v1/builds/race-car/preview.webp?v=32a32212e618",
        "model": "https://api.legoin.example/v1/builds/race-car/model.ldr?v=378de85baa5d"
      }
    },
    {
      "id": "idea_l8k3",
      "theme": "vehicles",
      "difficulty": "medium",
      "title": "Lokomotif",
      "summary": "Lokomotif merah dengan cerobong hitam.",
      "steps": [
        "Letakkan brick 2×4 merah sebagai badan lokomotif.",
        "Pasang brick 2×2 hitam di depan sebagai cerobong.",
        "Taruh plate 1×2 biru di belakang sebagai atap kabin.",
        "Tambahkan empat round 1×1 hitam di bawah sebagai roda."
      ],
      "used": [
        { "part": "brick-2x2", "color": "black", "quantity": 1 },
        { "part": "brick-2x4", "color": "red", "quantity": 1 },
        { "part": "plate-1x2", "color": "blue", "quantity": 1 }
      ],
      "missing": [
        { "part": "round-1x1", "color": "black", "quantity": 4 }
      ]
    },
    {
      "id": "idea_w5n0",
      "theme": "vehicles",
      "difficulty": "easy",
      "title": "Pesawat kecil",
      "summary": "Badan putih, sayap biru, ekor hijau.",
      "steps": [
        "Pakai brick 1×4 putih sebagai badan pesawat.",
        "Pasang plate 2×4 biru melintang di tengah sebagai sayap.",
        "Taruh slope 2×2 hijau di belakang sebagai ekor."
      ],
      "used": [
        { "part": "slope-2x2", "color": "green", "quantity": 1 }
      ],
      "missing": [
        { "part": "brick-1x4", "color": "white", "quantity": 1 },
        { "part": "plate-2x4", "color": "blue", "quantity": 1 }
      ]
    }
  ]
}
```

Errors: `400 invalid_request` for an unknown theme or locale, `404 not_found`, `409 no_pieces` when the scan has no pieces left to build with, `503 unavailable`, `504 timeout`. The web app never asks for ideas for a scan with no pieces.

### `GET /scans/{scanId}/ideas/{ideaId}`

`getIdea(scanId, ideaId)`: one idea from the scan's latest set, with that set, exactly as it was written. Ideas come back in the language they were written in, never translated, so the client compares `set.locale` with the page's language.

It answers `404 not_found` when the scan is gone or its latest set has no idea with this id: the set was replaced, or a correction dropped it. The client returns `null` and, as today, asks `GET /scans/{scanId}` to tell a replaced idea from a deleted scan.

```http
GET /v1/scans/scn_4kd2p9/ideas/idea_r4c8 HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```jsonc
{
  "idea": {
    "id": "idea_r4c8",
    "theme": "vehicles",
    "difficulty": "easy",
    "title": "Mobil balap kecil",
    // …the rest of the idea, as in the example above
  },
  "set": {
    "theme": "vehicles",
    "locale": "id",
    "items": [/* all three ideas of the set, as above */]
  }
}
```

## 7. Errors

Every error answers with one body:

```json
{
  "error": {
    "code": "invalid_request",
    "message": "changes[1].quantity must be from 1 to 20.",
    "field": "changes[1].quantity"
  }
}
```

- `code` is one of the codes below. Clients act on it alone.
- `message` is English, for logs and developers. People never see it: the web app writes its own message, in the page's language, from the code.
- `field` is only there for `invalid_request`: the path of the first field that failed.

| Code | Status | When | In the web app |
| --- | --- | --- | --- |
| `invalid_request` | 400 | The body, a field or a header fails validation: no photo part, an unknown theme, locale, part or color, a quantity out of range, an email or a password that breaks [the rules](#email-and-password). | A bug: the call rejects with a plain `Error`, and the screen shows its generic message. |
| `not_found` | 404 | No scan, job or idea with this id for this account. | `getScan` and `getIdea` return `null`; the other calls reject. |
| `unauthorized` | 401 | A route that needs a session has none: no `Authorization` header, one that isn't `Bearer <token>`, or a token that's unknown, expired or logged out. Sent with `WWW-Authenticate: Bearer`. | An `UnauthorizedError`: the data layer forgets the session, and the page sends the person to log in. `getAccount` returns `null`. |
| `invalid_credentials` | 401 | Logging in with an email that has no account, or with the wrong password. | `AuthError("invalid_credentials")`, which the log-in form shows. |
| `email_taken` | 409 | Registering an email that already has an account. | `AuthError("email_taken")` |
| `rate_limited` | 429 | Too many failed log-ins for one email. Sent with `Retry-After`. | `AuthError("rate_limited")` |
| `no_pieces` | 409, or in a failed job | The scan has no pieces to build with (ideas), or the models found none in the photo (scan). | `ApiError("no_pieces")` |
| `photo_too_large` | 413 | The photo is over 10 MB. | The client checks first and shows its own message (`PhotoProblem` `size`). |
| `photo_type` | 415 | The photo isn't JPEG, PNG or WebP, or can't be decoded. | The client checks first (`PhotoProblem` `type`). |
| `too_dark` | in a failed job | The photo is too dark to read. | `ApiError("too_dark")` |
| `too_blurry` | in a failed job | The photo is too blurry to read. | `ApiError("too_blurry")` |
| `timeout` | 504, or in a failed job | The scan or the ideas ran past their deadline. | `ApiError("timeout")` |
| `unavailable` | 503 | A model or the LLM provider is down or overloaded. It may send `Retry-After`. | `ApiError("network")` |
| `internal` | 500 | Anything else. | `ApiError("network")` |

A failed scan job isn't an HTTP error. The poll that reads it worked, so it answers `200` with the error in the job, in the same shape as the body above.

### ScanFailure reasons

`ScanFailure` in `domain.ts` is the reason an `ApiError` carries, which the screens turn into a message. Here's where each one comes from:

| `ScanFailure` | Comes from |
| --- | --- |
| `no_pieces` | A failed scan job with `no_pieces`, or `409 no_pieces` from a request for ideas. |
| `too_dark` | A failed scan job with `too_dark`. |
| `too_blurry` | A failed scan job with `too_blurry`. |
| `timeout` | A failed scan job with `timeout`, or `504 timeout` from a request for ideas. The client's own give-ups are `network`. |
| `network` | Never sent by the backend. The client uses it when it gets no usable answer: `fetch` rejects (offline, DNS, CORS), the status is 5xx with any other code, the code is `unavailable` or `internal`, the body can't be read, or polling gives up. |

The models decide which photo problem to report. The mock fails photos after the `detecting` stage, so `onStage` hears `detecting` before a photo failure; the real backend may fail at any stage.

### Timeouts

| What | Deadline | Then |
| --- | --- | --- |
| A scan job | 30 s after the upload | The job ends `failed` with `timeout`. |
| A request for ideas | 30 s | `504 timeout`. Nothing is saved. |
| The client, polling a job | 45 s with no ending, or 3 polls in a row with no answer | It sends `DELETE /scan-jobs/{jobId}` and rejects with `network`. |
| The client, waiting for ideas | 40 s | It aborts the request and rejects with `network`. |

These are the limits for failing, not the target. The target is under 15 seconds; see [Rules](#8-rules).

## 8. Rules

### Ideas come in the requested language, and are never translated

- `locale` is `en` or `id` ([ADR-0004](../decisions/0004-english-default-with-indonesian.md)). The agent writes each idea's `title`, `summary` and `steps` in it.
- In Indonesian, color names are Indonesian and part names stay in the hobby's English, as in the UI: "brick 2×4 merah", "plate 1×2 biru".
- A set is written once, in one language, and keeps it. The backend never rewrites a stored set into another language, and `GET /scans/{scanId}/ideas/{ideaId}` returns it as written. Asking in the other language writes a new set, which replaces the old one.
- Ids, themes, difficulty, part ids and color ids are the same in every language.

### Every idea passes `cek_inventori`

The PRD's `cek_inventori` check splits what a build needs into parts the person owns and parts they don't. The contract guarantees its result, whatever the agent does:

- The agent proposes each build with the full list of parts it needs, as part, color and quantity, and must call the `cek_inventori` tool with that list.
- The backend then runs the check itself on the final list, against the scan's inventory at the time of the request, and writes `used` and `missing` from its result. It never takes the model's own split. So a part the person doesn't have is always listed in `missing`, never counted as owned.
- The check is the one in legoin-web's `checkInventory`: for each line, `used` gets what the inventory covers, up to what's needed, and `missing` gets the rest. Each idea is checked against the whole inventory, independently of the other ideas.
- `used` never holds more of a line than the inventory does, and `used` plus `missing` is exactly what the build needs. `missing` may be empty. Lines are sorted the same way as the [inventory](#inventory).
- Ideas only use parts and colors from [the catalogue](#the-catalogue). An idea that needs anything else isn't returned; the agent writes another.

### Under 15 seconds from photo to ideas

- The PRD's target: under 15 seconds from photo to ideas. This draft sets it for a photo of up to 32 pieces, the most the detector's training photos hold.
- It's measured on the backend: from the end of the upload until the scan job is `done`, plus the time to answer the request for ideas. The time the person spends checking their pieces in between, and the phone's own network time, don't count.
- Budget: about 5 seconds for the scan and 10 for the ideas. The [deadlines](#timeouts) are where a call fails, not the target.

## 9. What stays out of the contract

**Demo options.** `?mock=fail-<reason>` and `?mockDelay=<ms>` are read from the page's URL by legoin-web's `src/lib/api/mock-backend.ts`. They're client-side only: they never reach the API, and the API has no way to fake a failure or a delay. The end-to-end suite keeps using them against the mock ([ADR-0007](../decisions/0007-end-to-end-smoke-tests.md)).

**Only in the mock:**

- its accounts and sessions, kept in `localStorage` with each account's scans;
- the four demo scans every new account starts with, and `resetMockData`;
- the made-up detections for a photo it doesn't know (`detectionsForPhoto`);
- the idea templates and their order: once unseen ideas run out, the mock offers the ones shown longest ago again, while the backend simply avoids `exclude`. The templates' text and parts are the backend's, which match its checked builds;
- the fake waits: 900 ms per stage, 1200 ms for ideas and 150 ms for reads and saves;
- the `localStorage` key `legoin.mock.scans.v1`.

**Client-side, with the real API too:**

- keeping the session in `localStorage` (`legoin.session.v1`), and sending people to log in when there's none or the API answers `401 unauthorized`;
- `inOrder`, which passes each stage on once;
- the scan handoff (`src/lib/scan-handoff.ts`), which shows a new scan's results in the same frame as the navigation;
- reading, downscaling and checking photos (`src/lib/photo.ts`); the backend checks the type and size again;
- everything about display: color names, the low-confidence flag and number formats. The API sends ids and plain numbers.

**Where the mock and the contract differ:**

- The mock's accounts exist only in the browser that made them.
- The mock's `photo.src` is a data URL or a path on the web app. The API's is a URL the backend serves.
- The mock's `preview` URLs are paths on the web app, to its copy of the library's pictures and models. The API's are URLs the backend serves.
- The mock writes ideas for a scan with no pieces, every part missing. The API answers `409 no_pieces`.
- For a scan that's gone, the mock's `updateDetections` and `getIdeas` reject with a plain `Error`. The API answers `404 not_found`; the screens treat both as a failure.
- The mock times out only when asked to, with `?mock=fail-timeout`. The API answers `timeout` whenever a scan or a request for ideas runs past its deadline.

## 10. Open questions

For the team. The contract works with any answer; the placeholders below only keep it complete until then.

1. **The LLM provider.** **Answered by [ADR-0009](../decisions/0009-anthropic-claude-writes-build-ideas.md): Anthropic's Claude.** The contract doesn't depend on one. Whatever is chosen needs tool calling for `cek_inventori`, good Indonesian, and an answer within the 10-second budget.
2. **Accounts and auth.** **Answered by [ADR-0011](../decisions/0011-accounts-with-email-and-password.md): accounts with an email and a password, required before scanning.** Is there a log-in? Yes. A session token in `Authorization` replaces the placeholder, the `Legoin-Device-Id` header, which anyone who had the id could use. See [Accounts](#4-accounts).
3. **Where photos are stored, for how long, and what else they're used for.** The contract only needs `photo.src` to load in a plain `<img>` while the scan exists, and deleting a scan to delete its photo. If photos are private, `src` has to be an unguessable or signed URL, since an `<img>` sends no `Authorization` header. ADR-0011 keeps ADR-0008's unguessable URLs.
4. **Whether history stays per device.** **Answered by [ADR-0011](../decisions/0011-accounts-with-email-and-password.md): no, history belongs to the account,** on every device it logs in on. Scans made per device before accounts move to the first account that logs in from that device ([Scans from before accounts](#scans-from-before-accounts)).

For the backend and ML owners:

5. **The photo size the models need.** The web app uploads its 1200 px copy. If part classification needs more pixels, the web app can send a larger one within 10 MB.
6. **Pieces outside the catalogue.** When the detector finds a piece the classifier can't place among the 13 parts and 10 colors, does the backend drop it, or send its nearest guess with a low confidence so the person is asked to check it?

## 11. Decisions in this draft

Technical choices this draft makes so the contract is complete. Each can be changed in review.

1. **Stages reach the client by polling a scan job, not by server-sent events.** Polling holds up on unsteady phone connections: a lost poll is retried and the scan carries on, where a dropped stream would lose it. It's plain JSON that the backend can serve without streaming and that `curl` can test, and the data layer's `inOrder` was written for a backend that polls. The cost is 20 to 30 small requests per scan, and a stage shown up to 500 ms late. The alternative is one `POST` that streams `stage` events and is cancelled by closing it.
2. **The scan job is a resource of its own.** `Scan` stays exactly as in `domain.ts`, with no status field, and a failed or cancelled scan never reaches history, as with the mock.
3. **Cancelling also deletes a scan that has just finished,** so a cancel always means nothing is saved.
4. **Ideas answer in one request,** since the agent's 10 seconds fit in an ordinary request and `getIdeas` has no stages to show.
5. **The app's part and color ids go on the wire, not Rebrickable ids,** because the code uses them everywhere. The catalogue maps each one to its Rebrickable or design id.
6. **The version is in the base URL (`/v1`), keys are camelCase, and optional fields are left out rather than `null`,** so answers parse straight into the types in `domain.ts`.
7. **Errors share one body with a `code`.** `network` stays the client's own reason, and `timeout`, `unavailable` and `internal` are new codes.
8. **Deletes always succeed:** `DELETE` answers `204` for a scan or job that's already gone, as the mock does.
9. **Corrections are all or nothing, and unknown detection ids are ignored,** as in the mock, so a retry or an undo is safe.
10. **`GET /scans` is wrapped (`{ "scans": [] }`) and has no paging,** since one account's history is small, and the wrapper lets paging come later without a new version.
11. **Ideas for a scan with no pieces answer `409 no_pieces`** instead of ideas with every part missing.
12. **The backend runs `cek_inventori` itself on the final parts list,** so the guarantee doesn't depend on the model calling the tool correctly.
13. **Times:** the 15-second target applies to photos of up to 32 pieces, with a budget of about 5 seconds for the scan and 10 for ideas; jobs and requests for ideas fail at 30 seconds; the client gives up at 45 seconds for a scan and 40 for ideas, and polls every 500 ms.
14. **Limits:** an `add` change adds 1 to 20 pieces, like the web app's add-piece form (`MAX_ADD_QUANTITY`), and `exclude` holds at most 50 ids.
15. **Pictures are two optional fields on `Idea`, `buildId` and `preview`,** sent only for checked builds from the backend's library ([ADR-0010](../decisions/0010-library-of-checked-builds.md)). Clients ignore fields they don't know, so this needs no new version. The URLs are absolute, like `photo.src`, and versioned, so they can be cached for good.
16. **Sessions are opaque tokens in `Authorization: Bearer`, which the client keeps in `localStorage`,** not cookies. The web app and the API are on different sites, where browsers block or limit cookies, and a token in a header needs no defence against cross-site requests. Photos and previews stay URLs that need no header ([ADR-0011](../decisions/0011-accounts-with-email-and-password.md)).
17. **Another account's scan, job or idea answers `404`, never `403`,** so an id tells nothing about other accounts.
18. **`POST /auth/logout` always answers `204`,** like the deletes in decision 8, so logging out never fails.

## 12. Changes from issue #7

The mock has grown since the issue was written, and the contract follows the code:

- Scanning is `POST /scan-jobs` and a job to poll (decisions 1 and 2), not `POST /scans` returning a scan id and status.
- Color ids are the app's ids, with Rebrickable ids in the catalogue (decision 5).
- The issue's `InventoryLine` is `PartCount`, and the inventory isn't sent: it's derived from the detections.
- New since the issue: the `restore` change for undo, `exclude` for "Try other ideas", the latest set kept on the scan with `lastTheme`, and `GET /scans/{scanId}/ideas/{ideaId}`.
- The issue lists `timeout` as a failure. The contract has it as an error code, and legoin-web's `ScanFailure` has it too since legoin-web#63, with its own message.

## 13. Picture builds

- Status: **Accepted**, with [ADR-0012](../decisions/0012-picture-to-lego-builds.md), in #25.

A person uploads a picture, such as a car, an animal or a house, and the agent designs a LEGO build of it from [the catalogue](#the-catalogue). The backend checks every build with BrickLayer and renders it. The person gets a picture of the build, an LDraw model, the parts, the steps, and, against a scan they choose, what they own and what they're missing.

A build takes about a minute and costs the backend money, so it runs as a job, like a scan, and it's limited per account and per day. Every route here needs a session ([Accounts](#4-accounts)), except the file URLs.

| Data layer (`src/lib/api`) | Endpoint | Answer |
| --- | --- | --- |
| `getPictureQuota()` | [`GET /picture-quota`](#get-picture-quota) | `PictureQuota` |
| `createPictureBuild(picture, locale, { onStage, signal })` | [`POST /picture-jobs`](#post-picture-jobs), then [`GET /picture-jobs/{jobId}`](#get-picture-jobsjobid) until it ends. [`DELETE /picture-jobs/{jobId}`](#delete-picture-jobsjobid) cancels. | A `PictureJob` that ends with a `PictureBuild` |
| `listPictureBuilds()` | [`GET /picture-builds`](#get-picture-builds) | `{ builds: PictureBuild[] }`, newest first |
| `getPictureBuild(id, { scanId })` | [`GET /picture-builds/{buildId}`](#get-picture-buildsbuildid) | `PictureBuild`, or 404, which the client turns into `null` |
| `deletePictureBuild(id)` | [`DELETE /picture-builds/{buildId}`](#delete-picture-buildsbuildid) | 204 |

### Picture build shapes

```ts
type PictureStage = "queued" | "designing" | "checking" | "rendering";

type PictureJob =
  | { id: string; status: "running"; stage: PictureStage }
  | { id: string; status: "done"; build: PictureBuild }
  | { id: string; status: "failed"; error: ApiErrorBody["error"] };

type PictureBuild = {
  id: string;
  createdAt: string; // ISO 8601, UTC: when the build was saved
  locale: Locale; // the language its title, summary and steps are written in
  title: string;
  summary: string;
  steps: string[];
  parts: PartCount[]; // every part the build needs, sorted like an inventory
  picture: Photo; // the uploaded picture, as kept
  preview: Preview; // the rendered build and its LDraw model
  plain?: true; // the title, summary and steps are the backend's plain ones
  check?: PictureCheck; // only when asked for with ?scanId=
};

type PictureCheck = {
  scanId: string;
  used: PartCount[]; // parts the person owns that the build uses
  missing: PartCount[]; // parts the build needs that the person doesn't have
};

type PictureQuota = {
  available: boolean; // whether POST /picture-jobs would take a picture now
  reason?: "off" | "limit" | "budget"; // why not, when it wouldn't
  used: number; // picture jobs this account started today (UTC)
  limit: number; // how many it may start a day
  resetsAt: string; // ISO 8601, UTC: the next midnight, when `used` goes back to 0
};
```

- **`parts`** are only catalogue parts and colors, at most 60 pieces in all.
- **`picture`** is the uploaded picture, upright, re-encoded as JPEG with its EXIF metadata removed. `src` is an absolute URL with an unguessable name, as `photo.src` is ([open question 3](#10-open-questions)).
- **`preview.image`** is an isometric picture of the build, a square WebP of 640 × 640 pixels on a transparent background, from the same angle as the library's. **`preview.model`** is the build as an LDraw file (`.ldr`), served as a download, with the same credit lines in its header as the library's. Both are absolute URLs with unguessable names, which load with no header for as long as the build exists. The client credits them as it does [library pictures](#preview).
- **`plain`** is there when a limit ended the job after the agent had made a valid build but before it wrote its own text: the build stands, its title and summary are plain, and its steps list the pieces level by level, from the bottom up. Otherwise it's left out.
- **`check`** is [`cek_inventori`](#every-idea-passes-cek_inventori) of `parts` against the scan's inventory at the time of the request, with the same rules as an idea's `used` and `missing`.

### What every picture build is

- **Physically valid,** as BrickLayer checks it: no two pieces collide, nothing floats, and every piece joins one connected assembly. A build that fails is never saved.
- **Designed with tools, never with code.** The agent places and removes catalogue pieces on a stud grid, asks for the checks and for renders, and finishes. The backend applies each call itself. Nothing the model writes is run.
- **Written in the requested language,** with the rules for [ideas](#ideas-come-in-the-requested-language-and-are-never-translated): Indonesian color names and the hobby's English part names in `id`. A build keeps the language it was written in.

**What's sent to the model provider:** the picture and the language. Never the account's id or email. The web app says so before the person uploads.

### `GET /picture-quota`

`getPictureQuota()`: whether this account can start a picture job now, and how many it has left today. The web app reads it before showing the upload.

```http
GET /v1/picture-quota HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```json
{ "available": true, "used": 1, "limit": 2, "resetsAt": "2026-11-26T00:00:00.000Z" }
```

`reason` says why `available` is `false`:

| `reason` | When |
| --- | --- |
| `off` | Picture builds aren't turned on on this backend. |
| `limit` | This account has started `limit` jobs today. |
| `budget` | The backend has spent its budget for the day. |

A full queue isn't a reason here: it passes in seconds, and `POST /picture-jobs` answers it with `Retry-After`.

### `POST /picture-jobs`

Uploads one picture as `multipart/form-data`, with two parts:

- `picture`: JPEG, PNG or WebP, at most 10 MB, with the same checks as a scan's photo;
- `locale`: `en` or `id`, the language to write the build's text in.

It answers `202` with a job waiting for its turn:

```http
POST /v1/picture-jobs HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
Content-Type: multipart/form-data; boundary=legoin

--legoin
Content-Disposition: form-data; name="picture"; filename="car.jpg"
Content-Type: image/jpeg

<JPEG bytes>
--legoin
Content-Disposition: form-data; name="locale"

en
--legoin--
```

```json
{ "id": "pjob_Q3v8tL2x", "status": "running", "stage": "queued" }
```

A job that's taken counts toward today's `limit` straight away. It's given back if the job then fails with `unavailable`, `internal` or `picture_budget`, or is cancelled before it leaves `queued`.

Errors:

- `400 invalid_request` when there's no `picture` part or `locale` isn't `en` or `id`;
- `413 photo_too_large` and `415 photo_type`, as for a scan's photo;
- `429 picture_limit` when this account has started `limit` jobs today, with `Retry-After` until midnight UTC;
- `503 picture_budget` when the backend has spent its budget for the day, with `Retry-After` until midnight UTC;
- `503 unavailable` when picture builds are off, with no `Retry-After`, or when the queue is full, with `Retry-After` in seconds.

### `GET /picture-jobs/{jobId}`

The job as it is now, as for a [scan job](#get-scan-jobsjobid). A running job reports its stage:

| Stage | What the backend is doing |
| --- | --- |
| `queued` | Waiting for the job before it to finish. One job runs at a time. |
| `designing` | The agent is designing the build. Most of the minute goes here. |
| `checking` | Checking the finished build once more, and counting its parts. |
| `rendering` | Rendering its picture and writing its LDraw model. |

The stage only moves forward. A finished job carries the saved build, which `GET /picture-builds` now lists too:

```jsonc
{
  "id": "pjob_Q3v8tL2x",
  "status": "done",
  "build": {
    "id": "pic_8KfD2mQx1Zr",
    "createdAt": "2026-11-25T08:41:05.000Z",
    "locale": "en",
    "title": "Little Red Car",
    "summary": "A red car with four black wheels, a sloped cabin with blue side windows, yellow headlights at the front and orange taillights at the back.",
    "steps": [
      "Stand four black round 1×1 bricks as wheels at the corners, 2 studs in from each end.",
      "Lay five black 2×4 plates on top to make a chassis 10 studs long and 4 studs wide."
      // …
    ],
    "parts": [
      { "part": "brick-2x4", "color": "red", "quantity": 9 },
      { "part": "plate-2x4", "color": "black", "quantity": 5 }
      // …
    ],
    "picture": { "src": "https://api.legoin.example/v1/picture-files/n4Qk0vXb7cT2mZr9LwYe1A.jpg", "width": 900, "height": 600 },
    "preview": {
      "image": "https://api.legoin.example/v1/picture-files/Hq7bW3pZ0sKd5yRf2LmNcQ.webp",
      "model": "https://api.legoin.example/v1/picture-files/Vt8cX1nB6qLs4wJe9KdRaQ.ldr"
    }
  }
}
```

A failed job carries the error, and nothing was saved:

```json
{
  "id": "pjob_Q3v8tL2x",
  "status": "failed",
  "error": { "code": "picture_unusable", "message": "There's no clear subject in the picture to build." }
}
```

| Code in a failed job | When |
| --- | --- |
| `picture_unusable` | The agent found no clear subject to build: a blank, abstract, very dark or blurry picture, or only text. |
| `picture_refused` | The model declined the picture, or the agent found it unsuitable for a family building app. |
| `no_valid_build` | A limit on the job's cost, model calls or time ended it before the agent made a valid build. |
| `picture_budget` | The backend's daily budget ran out before the job could start. |
| `timeout` | The job ran past its deadline. |
| `unavailable` | The model provider couldn't be reached or was overloaded. |
| `internal` | Anything else. |

The rules of a scan job hold: a failed job still answers `200`, an ended job is kept for at least 10 minutes, and only an unknown job is `404 not_found`.

### `DELETE /picture-jobs/{jobId}`

Cancels the job and answers `204` with no body, as for a [scan job](#delete-scan-jobsjobid). The backend stops the job at once and saves nothing; if the job had already finished, it deletes the build the job saved. An unknown or forgotten job also answers `204`. What the job spent before the cancel still counts toward the daily budget.

### `GET /picture-builds`

`listPictureBuilds()`: this account's picture builds, newest first by `createdAt`, wrapped like [`GET /scans`](#get-scans). None answers `{ "builds": [] }`. The builds carry no `check`.

### `GET /picture-builds/{buildId}`

`getPictureBuild(id, { scanId })`: one build. With `?scanId=`, it adds `check` against that scan, as the scan is now:

```http
GET /v1/picture-builds/pic_8KfD2mQx1Zr?scanId=scn_4kd2p9 HTTP/1.1
Authorization: Bearer lgn_pY5h2PcG9NnqPsZk4j-mzx5D7dFTpfR8_jUFioj9cOg
```

```jsonc
{
  "id": "pic_8KfD2mQx1Zr",
  // …the rest of the build, as above
  "check": {
    "scanId": "scn_4kd2p9",
    "used": [{ "part": "brick-2x4", "color": "red", "quantity": 1 }],
    "missing": [
      { "part": "brick-2x4", "color": "red", "quantity": 8 },
      { "part": "plate-2x4", "color": "black", "quantity": 5 }
      // …
    ]
  }
}
```

An unknown build, or another account's, answers `404 not_found`, and the client returns `null`. A `scanId` that isn't one of this account's scans leaves `check` out, since the build itself is there; the client compares `check.scanId` with the scan it asked about.

### `DELETE /picture-builds/{buildId}`

`deletePictureBuild(id)`: deletes the build with its picture, preview and model, and answers `204` with no body. An unknown build also answers `204`.

### Errors and limits for picture builds

These codes are new, and only picture builds use them:

| Code | Status | When | In the web app |
| --- | --- | --- | --- |
| `picture_limit` | 429 | This account has started its picture jobs for today. Sent with `Retry-After`. | A message with when it resets, from `GET /picture-quota` |
| `picture_budget` | 503, or in a failed job | The backend has spent its budget for the day. Sent with `Retry-After`. | A message to try again tomorrow |
| `picture_unusable` | in a failed job | No clear subject to build. | A message to try a clearer picture of one thing |
| `picture_refused` | in a failed job | The model declined the picture, or the agent found it unsuitable. | A message to try another picture |
| `no_valid_build` | in a failed job | A limit ended the job before a valid build. | A message to try again, or a simpler picture |

| What | Deadline | Then |
| --- | --- | --- |
| A picture job, from leaving `queued` | 6 minutes | The job ends `failed` with `timeout`. |
| The client, polling a picture job | Every second; 10 minutes with no ending, or 3 polls in a row with no answer | It sends `DELETE /picture-jobs/{jobId}` and rejects with `network`. |

The backend's own limits (the daily budget, the jobs per account a day, the cost and time a job may take, and the queue's length) are its configuration, which [ADR-0012](../decisions/0012-picture-to-lego-builds.md) sets out. The client learns the ones it needs from `GET /picture-quota` and `Retry-After`.

### Idea previews

Ideas the agent writes itself get a picture too, built after the ideas are answered by the same engine as picture builds ([ADR-0012](../decisions/0012-picture-to-lego-builds.md), "Idea previews"). `POST /scans/{scanId}/ideas` answers as it does today, within its deadline; a preview never slows it down or fails it. Library ideas keep the `buildId` and `preview` they've always had.

One optional field joins `Idea` ([Shapes](#3-shapes)):

```ts
type Idea = {
  // …every field in section 3, and:
  previewStatus?: "pending" | "on_open" | "ready" | "unavailable";
};
```

| `previewStatus` | What it means | What the client does |
| --- | --- | --- |
| left out | The backend makes no preview for this idea: it's a library idea with its own `preview`, or previews are off. | As today. |
| `pending` | The preview is waiting for its turn or being built. | Shows that a picture is coming, and reads the idea again every 3 seconds, with `GET /scans/{scanId}` on the ideas list or `GET /scans/{scanId}/ideas/{ideaId}` on the idea's page, until the status changes. |
| `on_open` | The backend builds the preview when this idea's page is opened. | Nothing on the list. `GET /scans/{scanId}/ideas/{ideaId}` starts the build and already answers `pending`. |
| `ready` | The preview is built. | Shows `preview`. |
| `unavailable` | No preview will come: a limit or the budget was reached, or the build couldn't be made. | Shows the idea as text, as today. |

When a preview is `ready`, **the build is the idea:**

- `buildId` and `preview` are there. A built preview's `buildId` starts with `gen_`, and its `preview` URLs are unguessable files of this account's, like a picture build's.
- `steps` and `summary` are rewritten to match the build, in the set's language; the title stays.
- `used` and `missing` come from [`cek_inventori`](#every-idea-passes-cek_inventori) of the build's own parts against the scan's inventory at that moment. The build may use more or fewer of each of the idea's parts and colors than it listed, but no others.

So the picture, the parts and the steps never disagree. A client that kept the idea from before reads it again.

Previews belong to the set they were made for:

- **Asking for ideas again**, or a correction that drops the scan's `ideas`, cancels the old set's previews that haven't finished. Their builds are deleted with the old set.
- **Deleting the scan** deletes its previews.
- **Previews count toward the backend's daily budget** for picture builds. Each account also has a daily number of previews; past it, or past the budget, a new idea's `previewStatus` is `unavailable` straight away.

**Decisions in this section,** for review:

1. **A picture build is a job,** like a scan, not one long request: it takes about a minute, the person sees its stages, and a dropped connection doesn't lose it.
2. **A picture build isn't an `Idea`.** It has no theme or difficulty, it belongs to the account rather than to a scan, and its parts aren't checked until the person picks a scan, so it gets shapes of its own.
3. **`check` is worked out on each read,** against the scan as it is then, so a correction to the scan shows straight away and nothing has to be kept in step.
4. **The quota has its own route,** so the web app can say what's left before the person picks a picture, rather than after an upload is refused.
5. **A preview is a status on the idea, read by polling the routes the client already uses,** not a job of its own: the ideas list already reads `GET /scans/{scanId}`, and the idea's page `GET /scans/{scanId}/ideas/{ideaId}`.
6. **A built preview replaces the idea's steps, summary, `used` and `missing`,** so the words and the picture always describe the same build.
