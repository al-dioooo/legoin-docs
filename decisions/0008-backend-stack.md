# ADR-0008: Backend in Python with FastAPI, Models from a Pinned legoin-ml Release, SQLite and Local Photos

- Status: Proposed
- Date: 2026-11-25
- Deciders: Legoin team
- Issue: #15

## Context and Problem

The [API contract](../api/contract.md) (draft v0.1, #7) says what the backend answers, and legoin-ml release [v0.2.0](https://github.com/al-dioooo/legoin-ml/releases/tag/v0.2.0) holds the three stages it runs: the YOLO11n detector, the MobileNetV3-Large part classifier and the provisional color parameters, with every setting the backend must match. The backend doesn't exist yet. No LLM provider is chosen and the demo host isn't decided. What should the backend be built with, how does it run the models, and where does it keep scans and photos?

## Decision Drivers

- **The models are Python.** They load with Ultralytics, PyTorch and torchvision, and the color step is NumPy, scikit-image and scikit-learn code from notebook 03. The release notes name the exact versions ([ADR-0005](0005-ml-repository-and-model-releases.md)).
- **Follow the contract to the letter.** That means camelCase keys, optional fields left out rather than `null`, one error body, scan jobs served by polling, and deadlines of 30 seconds. legoin-web builds its HTTP client against the same document.
- **Fast CI without the models.** Unit tests and the required check shouldn't install torch or download weights.
- **A CPU is enough.** The models were trained and timed on a 16-vCPU server with no GPU. There's no budget for a GPU host or paid inference.
- **No keys or paid services** while the LLM provider is an open question.
- **Few moving parts.** A student team runs this for a course demo: one process, one data folder, nothing else to operate.
- **Any host.** The host is a team decision still to come, so the service has to run from a container image.
- **The same conventions as legoin-ml:** an issue first, small pull requests, Conventional Commits, CI on every pull request and a required check.

## Considered Options

Language and framework:

1. Python 3.12 with FastAPI, uvicorn and pydantic v2
2. Python with Flask, or Django REST framework
3. TypeScript, next to legoin-web, calling a separate Python service for the models

Running the models:

1. PyTorch and Ultralytics at the release's versions, from CPU wheels
2. Export to ONNX and run ONNX Runtime
3. Hosted inference: Roboflow, Replicate, Hugging Face endpoints or Modal

Scan jobs:

1. An in-process worker, with jobs kept in memory
2. A task queue: Celery or RQ, with Redis

Storage:

1. SQLite and the local disk, in one data directory
2. PostgreSQL and object storage, such as S3, R2 or Supabase

Ideas, until an LLM provider is chosen:

1. legoin-web's idea templates, ported to Python behind an interface an LLM can plug into
2. Pick a provider now

## Decision

Build the backend in a new public repository, [`al-dioooo/legoin-api`](https://github.com/al-dioooo/legoin-api), with **Python 3.12, FastAPI, uvicorn and pydantic v2**. It runs **the pinned v0.2.0 models with PyTorch on a CPU**, runs **scan jobs in an in-process worker**, and keeps **scans in SQLite and photos on the local disk**. It writes **ideas from templates** behind an interface an LLM provider can plug into later.

### Language, framework and tooling

- **FastAPI** on **uvicorn**. Its pydantic v2 models mirror the contract's shapes, it validates requests, and it serves an OpenAPI page for free.
- **uv** manages Python and the dependencies with `pyproject.toml` and a committed `uv.lock`.
- **Ruff** lints and formats. **pytest** tests through FastAPI's `TestClient`.
- **CI** runs one job, "Lint and test", on every `pull_request` with no base filter, so stacked pull requests get CI too. It runs `ruff check`, `ruff format --check` and `pytest` on the stand-in pipeline described below. That job becomes the required check on `main`.
- **Process.** The repository follows legoin-ml:
  - `AGENTS.md`, with `CLAUDE.md` pointing to it;
  - a pull request template;
  - one bootstrap commit straight to `main`;
  - branch protection with one approving review, strict status checks and the CI job as the required check, with admins not enforced.

### Models, from a pinned release

- The backend downloads its model files from a legoin-ml release tag. The tag is configuration (`LEGOIN_MODEL_TAG`, default `v0.2.0`).
- Each file's SHA-256 from the release notes is written in the code, per tag, and checked after every download and on every start. A file that doesn't match is refused. Moving to a new tag is therefore a reviewed code change, which copies the new settings and hashes from its release notes, as ADR-0005 expects.
- Files are cached in the data directory (`models/<tag>/`) and never committed. `*.pt` is in `.gitignore`.

The inference libraries are pinned to the release notes' versions:

- Ultralytics 8.4.190;
- torch 2.15.0 and torchvision 0.29.2, as CPU wheels from the PyTorch index;
- scikit-learn 1.9.1 and scikit-image 0.26.0;
- Pillow.

They are an optional dependency group (`models`), so CI never installs them. Ultralytics' default `opencv-python` is swapped for `opencv-python-headless`, so a server or a slim container image needs no `libGL`.

### The pipeline, as the release notes say

For each photo:

1. **Upright first.** Apply the EXIF orientation, then convert to RGB. Every later step, and the stored photo, uses that upright image.
2. **Detect** with YOLO at `imgsz` 640, `conf` 0.65, `iou` 0.7 and `max_det` 300. Boxes become the contract's fractional `Box`.
3. **Crop** each box widened the way Ultralytics' `save_one_box` does it (1.02× plus 10 px, clipped to the photo).
4. **Classify the part.** Letterbox the crop to 224 px on a `(124, 116, 104)` square, then apply ImageNet normalization and `softmax(logits / 0.7493)` over the 14 outputs. Send the argmax of outputs 0–12; its probability is the part confidence. `other` is never sent.
5. **Read the color.** Port `warna_kepingan`, `jarak` and `putuskan` from section 5 of notebook 03 at `bfee134`, with the parameters in `legoin_warna.json`.
6. **Confidence** is the lowest of the box, part and color scores.

A detected piece is never dropped. That's the release's proposed answer to the contract's open question 6, and the backend follows it until the team decides.

**Photo problems.** When the detector finds nothing, the job fails with `no_pieces`. Two checks then pick a better reason, but only for a photo with no pieces: a very dark photo fails with `too_dark` instead, and one with almost no edges fails with `too_blurry`. Their thresholds are provisional until labeled photos exist. These checks never apply to a photo with pieces in it.

**The stand-in pipeline.** `LEGOIN_PIPELINE` chooses between `models` and `stub`.

- The stub gives deterministic detections worked out from the photo's pixels. It needs no torch and no weights, so unit tests and CI run without them.
- It's the default outside the container, so a fresh clone runs without downloading anything. The container image defaults to `models`.
- `GET /health` reports which pipeline is running, and the service logs a warning at start-up when it serves the stub.

### Scan jobs

- `POST /scan-jobs` checks the photo, keeps it in memory and queues the job. A thread pool runs jobs one at a time by default, since torch already uses every core. A queued job reports `detecting`.
- The worker moves the job through `detecting`, `classifying` and `coloring`, and clients poll for it.
- **Cancel.** `DELETE` marks the job cancelled. The worker checks between stages and saves nothing. If the job had already finished, the delete removes the scan it saved.
- **Deadline.** A job still running 30 seconds after the upload ends `failed` with `timeout`. A late result is thrown away, never saved.
- Ended jobs stay in memory for 10 minutes, then answer `404`.
- **Photos.** The photo is written to disk only when the scan is saved, so a failed or cancelled job leaves nothing behind.

### Storage

- **SQLite.** One file in the data directory holds scans with their detections, `lastTheme` and the latest idea set. It also remembers every idea written for a scan, which `exclude` needs. Every row is keyed by the `Legoin-Device-Id` header, the contract's placeholder, and every query filters by it.
- **Photos.** They live in the data directory too, under `photos/`, and the API serves them.
- **The data directory** is `LEGOIN_DATA_DIR`. It's the only state the service has, so a host has to keep it on a persistent disk.

### Provisional answers to open questions 3 and 4

These are what the backend does until the team decides. The contract works with either answer and doesn't change.

**3. Where photos are stored, for how long, and what else they're used for.**

- **Where.** On the backend's own disk, in `LEGOIN_DATA_DIR/photos/`.
- **What's stored.** The upright photo, re-encoded as JPEG with its EXIF metadata removed, so no location or camera data is kept.
- **Its URL.** `photo.src` is an absolute URL on the API with a random, unguessable 128-bit file name. A plain `<img>` loads it with no header, and it can't be guessed from the scan id or the device id.
- **For how long.** Exactly as long as the scan. Deleting the scan deletes the photo, and a failed or cancelled job never writes one.
- **Other uses.** None. Photos aren't used for training, shared or copied anywhere else. Adding them to the training data would need people's consent and a decision of its own.

**4. Whether history stays per device.**

- **Yes, per device.** Each device sees only its own scans, keyed by `Legoin-Device-Id`, and nothing links one device to another.
- **Server-side.** The scans live in the backend's SQLite, not in the browser, so clearing the browser's storage loses the device id, not the scans. Those scans become unreachable.
- **Kept until deleted.** Scans are kept until the person deletes them; there's no expiry yet.
- **Across devices.** History across devices waits for accounts (open question 2).

Questions 1 (the LLM provider), 2 (accounts and auth), 5 (photo size) and 6 (pieces outside the catalogue) stay open.

### Ideas

- **The interface.** An `IdeaGenerator` interface proposes builds: theme, difficulty, title, summary, steps and the full list of parts each build needs.
- **For now, templates.** The only generator ports legoin-web's 18 idea templates, six per theme in English and Indonesian. It prefers templates that use more of the inventory and haven't been shown for the scan, and it avoids `exclude`.
- **Later, an LLM.** A provider plugs in behind the same interface. It needs no key until then.
- **`cek_inventori`** is ported from legoin-web's `checkInventory` to Python, and it's also described as a tool an LLM can call.

The service around the generator enforces the contract's rules, whichever generator runs:

- it drops any build with a part or color outside the catalogue and asks for another;
- it runs `cek_inventori` itself on each build's final parts list, against the scan's inventory at the time of the request, and writes `used` and `missing` from its result;
- it answers `409 no_pieces` for a scan with no pieces, and `504 timeout` after 30 seconds;
- it keeps the new set as the scan's `ideas` and sets `lastTheme`;
- a correction that changes the inventory drops the kept set.

### Configuration

Configuration comes from environment variables, with defaults that run locally:

| Variable | Default | What it sets |
| --- | --- | --- |
| `LEGOIN_PIPELINE` | `stub` (`models` in the container) | Which pipeline runs scans |
| `LEGOIN_MODEL_TAG` | `v0.2.0` | The legoin-ml release to download models from |
| `LEGOIN_DATA_DIR` | `./data` | The SQLite file, photos and cached models |
| `LEGOIN_CORS_ORIGINS` | `http://localhost:3000,https://legoin-web.vercel.app` | Origins allowed to call the API |
| `LEGOIN_PUBLIC_URL` | taken from each request | The absolute base for `photo.src`, for a host behind a proxy |

CORS allows exactly the listed origins, the methods `GET`, `POST`, `PATCH` and `DELETE`, and the headers `Content-Type` and `Legoin-Device-Id`. No wildcard is used, so a preview deployment's origin has to be added to the list.

### Container image, but no deployment

The repository has a CPU `Dockerfile`:

- the base image is `python:3.12-slim`;
- uv installs the locked dependencies with the `models` group;
- the model files are downloaded and checked at build time, so the container starts without network access;
- the service runs as a non-root user, as one uvicorn process, with the data directory as a volume.

Nothing is deployed. The host stays a team decision, and this ADR only says what a host needs: a persistent disk for the data directory, enough memory for torch, and a single instance.

## Consequences

- Positive: the backend runs the released models with the exact libraries and settings they were made with. The SHA-256 check means it can't load a file that differs from the release.
- Positive: CI and unit tests run in seconds with no torch, weights or network, and the stub gives the web team a backend to build against before the models run anywhere.
- Positive: a CPU is enough. On the project's 8-core server, legoin-api scans a real 31-piece photo in about 0.4 s from the end of the upload, under a tenth of the contract's 5-second budget, so a smaller CPU still has room.
- Positive: one process and one folder. There's no queue, database server, bucket or key to set up, and the whole service runs from a container or from `uv run`.
- Positive: an LLM provider can be added later behind `IdeaGenerator` without changing the contract or the routes. `cek_inventori` and the catalogue check already guard whatever it writes.
- Negative: jobs live in memory, so the service must run as **one process**, with no extra uvicorn workers and no second instance. A restart forgets running jobs; the client then gives up after 3 failed polls and reports `network`.
- Negative: SQLite and the local disk tie the data to one machine. Free hosts with ephemeral disks lose every scan and photo on a redeploy unless they mount a persistent volume. Moving to PostgreSQL and object storage later means a migration.
- Negative: torch makes the container image large: about 2.0 GB unpacked and 0.6 GB to download, measured on legoin-api's image. The container used about 425 MB of memory idle and 550 MB after scanning, so a free tier with 512 MB won't fit.
- Negative: model settings are copied by hand from release notes into code, and a new tag means a code change (already true under ADR-0005).
- Negative: the template ideas are a stand-in. Only six exist per theme, so after six "Try other ideas" in one theme the backend has to offer excluded ones again, where an LLM would write new ones.
- Negative: `too_dark` and `too_blurry` come from simple checks with uncalibrated thresholds, until labeled photos exist.

## Pros and Cons of the Options

### Python with FastAPI

- Pro: the same language as the models, so inference runs in-process with the release's code and libraries.
- Pro: pydantic models mirror the contract's shapes and validate every request, and the OpenAPI page documents the API for the web team.
- Con: one more language next to legoin-web's TypeScript, and the shapes are written twice, in `domain.ts` and in pydantic.

### Python with Flask or Django REST framework

- Pro: well known. Django brings an ORM, migrations and an admin.
- Con: Flask has no built-in validation or schema, so every contract rule is hand-written. Django is much more framework than a dozen endpoints need.

### TypeScript, plus a Python model service

- Pro: shares types with legoin-web.
- Con: two services to build, deploy and connect for one demo, and the models still need Python.

### PyTorch and Ultralytics at the release's versions

- Pro: exactly what the models were made and checked with, so results match the notebooks.
- Con: a large install and image, and more memory than an exported model.

### ONNX Runtime

- Pro: a smaller image, less memory and often faster on a CPU.
- Con: the release ships `.pt` files, so an export step and a check that the exported models give the same answers would come first. Worth revisiting if the image or memory blocks a host.

### Hosted inference

- Pro: no torch in the image.
- Con: keys, cost and network time inside the 5-second scan budget, and a dependency on a third party for a course demo.

### In-process worker

- Pro: nothing to run besides the API, and stages, cancelling and deadlines are plain Python.
- Con: one process only, and jobs don't survive a restart.

### Celery or RQ with Redis

- Pro: jobs survive restarts and scale across workers.
- Con: Redis and a worker process to deploy and watch, for a demo that scans one photo at a time.

### SQLite and the local disk

- Pro: nothing to set up, and one folder to back up or mount.
- Con: tied to one machine and one process, and needs a persistent disk.

### PostgreSQL and object storage

- Pro: survives redeploys on any host and scales out.
- Con: two more services and their credentials before the host is even chosen.

### Templates behind an interface

- Pro: works today with no key or cost, matches what the web app's mock shows, and keeps the provider question open.
- Con: a fixed set of ideas, not written for the person's pieces.

### Picking a provider now

- Pro: real ideas written for each inventory.
- Con: takes a team decision (open question 1) out of the team's hands, and needs keys and a budget.

## References

- [API contract](../api/contract.md), draft v0.1, and its open questions 3 and 4
- [ADR-0004](0004-english-default-with-indonesian.md): ideas in the requested language
- [ADR-0005](0005-ml-repository-and-model-releases.md): models in legoin-ml, weights as releases
- legoin-ml release [v0.2.0](https://github.com/al-dioooo/legoin-ml/releases/tag/v0.2.0) and its settings, and [notebook 03 at `bfee134`](https://github.com/al-dioooo/legoin-ml/blob/bfee134/notebooks/03-kmeans-cielab-brick-color.ipynb), section 5
- legoin-web `src/lib/inventory.ts` (`checkInventory`, `toInventory`) and `src/lib/mock/fixtures.ts` (`IDEA_TEMPLATES`)
- PRD Legoin v0.2, section 7 (architecture and the `cek_inventori` agent)
- [legoin-api](https://github.com/al-dioooo/legoin-api)
