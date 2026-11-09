# Legoin API Contract

- Status: Draft, under review in #7
- Version: 0.1
- Date: 2026-11-09
- Issue: #7
- Source of the shapes: `src/lib/domain.ts` and `src/lib/api` in [legoin-web](https://github.com/al-dioooo/legoin-web), on `main` at `a05024d`

This is the contract between legoin-web and the Python backend. The backend runs the models (detection, then part classification and color, as in [legoin-ml](https://github.com/al-dioooo/legoin-ml)) and an LLM agent that suggests builds. Both sides build against this document, independently.

In legoin-web, screens read and write only through the data layer, `src/lib/api`. Today a browser-side mock serves it from `localStorage`. Setting `NEXT_PUBLIC_API_URL` will switch it to an HTTP client for this API. Every function in the data layer has an endpoint here, and every shape matches `domain.ts`, so the switch changes nothing outside `src/lib/api`.

Parts the team hasn't decided yet are marked **Open question** and collected in [Open questions](#8-open-questions). Technical choices this draft makes on its own are collected in [Decisions in this draft](#9-decisions-in-this-draft), so a reviewer can push back on them.

## 1. Overview

| Data layer (`src/lib/api`) | Endpoint | Answer |
| --- | --- | --- |
| `createScan(photo, { onStage, signal })` | [`POST /scan-jobs`](#post-scan-jobs), then [`GET /scan-jobs/{jobId}`](#get-scan-jobsjobid) until it ends. [`DELETE /scan-jobs/{jobId}`](#delete-scan-jobsjobid) cancels. | A `ScanJob` that ends with a `Scan` |
| `getScan(id)` | [`GET /scans/{scanId}`](#get-scansscanid) | `Scan`, or 404, which the client turns into `null` |
| `listScans()` | [`GET /scans`](#get-scans) | `{ scans: Scan[] }`, newest first |
| `updateDetections(id, changes)` | [`PATCH /scans/{scanId}/detections`](#patch-scansscaniddetections) | The updated `Scan` |
| `deleteScan(id)` | [`DELETE /scans/{scanId}`](#delete-scansscanid) | 204 |
| `getIdeas(scanId, theme, locale, { exclude })` | [`POST /scans/{scanId}/ideas`](#post-scansscanidideas) | An `IdeaSet` of 3 ideas |
| `getIdea(scanId, ideaId)` | [`GET /scans/{scanId}/ideas/{ideaId}`](#get-scansscanidideasideaid) | `StoredIdea`, or 404, which the client turns into `null` |

A visit goes: upload a photo and follow the scan's stages, check and correct the pieces, ask for ideas in a theme, open one idea, and come back to the scan later from history.

## 2. Conventions

- **Base URL.** `NEXT_PUBLIC_API_URL` is the base, version included, e.g. `https://api.legoin.example/v1`. Paths below are relative to it. A breaking change gets a new version (`/v2`).
- **JSON.** Requests and answers are JSON in UTF-8, except the photo upload, which is `multipart/form-data`. Keys are camelCase, with the same names as in `domain.ts`.
- **Optional fields** are left out when they have no value, never sent as `null`. The exceptions are a detection's `box` and `confidence`, which are always present and `null` for a piece added by hand.
- **Unknown fields.** A client ignores fields it doesn't know, so the backend can add an optional field within a version.
- **Times** are ISO 8601 in UTC, ending in `Z`, e.g. `2026-11-09T03:12:45.120Z`. Sorting them as strings sorts them by time.
- **Ids** are opaque strings the backend makes: URL-safe (`A–Z a–z 0–9 _ -`), at most 64 characters. The client never parses one. The only id the client sends back unprompted is a detection's, in a `restore` change.
- **Whose scans.** Every request carries a `Legoin-Device-Id` header, and the backend only shows a device its own scans. This is a placeholder until the team decides on accounts. See [Open questions](#8-open-questions) 2 and 4.
- **CORS.** The data layer runs in the browser, in Client Components, so the browser calls the API directly. The API allows the web app's origins, the methods `GET`, `POST`, `PATCH` and `DELETE`, and the request headers `Content-Type` and `Legoin-Device-Id`.
- **Errors** share one body. See [Errors](#5-errors).

## 3. Shapes

Written as TypeScript because `domain.ts` is; the backend can mirror them with Pydantic models. Everything here is in `domain.ts` or `bricks.ts` with the same names and fields, except `ScanJob` and `ApiErrorBody`, which only exist on the wire.

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
};

type IdeaSet = { theme: Theme; locale: Locale; items: Idea[] };

type StoredIdea = { idea: Idea; set: IdeaSet };

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

type ApiErrorBody = {
  error: { code: ErrorCode; message: string; field?: string };
};
```

`ErrorCode` is listed in [Errors](#5-errors), and `BrickPartId` and `BrickColorId` in [the catalogue](#the-catalogue).

### Photo

- `src` is an absolute URL that a plain `<img>` can load, with no extra headers, for as long as the scan exists.
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

## 4. Scans

### Scanning a photo

`createScan(photo, { onStage, signal })` uploads a photo, reports the pipeline's stages as they happen, and resolves with the saved scan. Over HTTP that's a scan job the client polls:

1. `POST /scan-jobs` uploads the photo and answers `202` with a running job.
2. The client asks `GET /scan-jobs/{jobId}` every 500 ms and passes each job's `stage` to `onStage`. The data layer's `inOrder` already drops repeats and steps back, and a stage that changed between two polls is skipped.
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
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
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
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
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
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
```

```http
HTTP/1.1 204 No Content
```

### `GET /scans/{scanId}`

`getScan(id)`: one scan, with its detections and, once ideas were asked for, its `lastTheme` and latest `ideas`. The answer is the `scan` from the finished job above, as it is now. An unknown scan, or one from another device, answers `404 not_found`, and the client returns `null`.

```http
GET /v1/scans/scn_4kd2p9 HTTP/1.1
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
```

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{ "error": { "code": "not_found", "message": "No scan scn_4kd2p9." } }
```

### `GET /scans`

`listScans()`: this device's scans, newest first by `createdAt`, as full `Scan` objects. They're wrapped in an object so paging can be added later without a new version; this version has no paging. No scans answers `{ "scans": [] }`.

```http
GET /v1/scans HTTP/1.1
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
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

This request recolors the flagged piece and adds a blue plate the detector missed. The scan had ideas, and the inventory changed, so they're gone; `lastTheme` is still there:

```http
PATCH /v1/scans/scn_4kd2p9/detections HTTP/1.1
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
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
Legoin-Device-Id: 3b0e6f1c-5d2a-4f7e-9c41-8a2d7e5b9f10
```

```http
HTTP/1.1 204 No Content
```
