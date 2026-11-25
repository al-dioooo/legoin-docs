# ADR-0011: Accounts with Email and Password, Required Before Scanning

- Status: Proposed
- Date: 2026-11-25
- Deciders: Legoin team
- Issue: #23

## Context and Problem

The [API contract](../api/contract.md) left accounts open (open question 2). Until now, every request carried `Legoin-Device-Id`, a random UUID the browser makes once and keeps in `localStorage`, and [ADR-0008](0008-backend-stack.md) keyed every scan by it. It keeps devices apart, but it isn't security: anyone who has the id sees its scans. Clearing the browser loses them, and history can't follow a person to another device (open question 4). The team has decided to add a log-in, with an email and a password, before anyone can scan. How are passwords kept, what does a session look like on the wire, which routes need one, and what happens to scans made under a device id?

## Decision Drivers

- **The team's choice:** email and password, required before scanning, decided on 2026-11-25.
- **Scans only their owner can reach,** on any device the owner logs in on.
- **No third party yet.** There's no email service and no OAuth client, so there are no keys, no cost and no console to set up. That rules out a reset email, an email check and magic links for now.
- **Passwords kept safely.** A slow, memory-hard hash, never logged and never stored as typed.
- **ADR-0008 still holds:** one process, SQLite and the local disk, no new service, and CI without the models. No new environment variable, if possible.
- **The browser calls the API directly, from another site.** legoin-web runs on `legoin-web.vercel.app` and the API on a host of its own, so a cookie the API sets is a third-party cookie to the browser.
- **`<img>` loads photos and previews** and sends no header.
- **Small enough for a course demo:** quick to build, easy to explain, easy to run.

## Considered Options

Signing in:

1. An email and a password
2. Google sign-in: the API checks Google's ID token and makes its own session
3. A magic link sent by email

When it's needed:

1. Before scanning: every scan belongs to an account
2. Optional: guests scan with the device id, and their scans move to the account when they log in

The session:

1. An opaque token sent as `Authorization: Bearer`, kept by the client in `localStorage`
2. An `HttpOnly` cookie
3. A signed JWT

The password hash:

1. Argon2id, with `argon2-cffi`
2. bcrypt
3. scrypt or PBKDF2 from Python's standard library

Photos:

1. Unguessable URLs, as now
2. Signed URLs that expire
3. Fetched with the token and shown from a `blob:` URL

Scans made under a device id:

1. Moved, once, to the account that logs in from that device
2. Left behind

## Decision

Accounts have **an email and a password, hashed with Argon2id**. Logging in gives **an opaque session token, sent as `Authorization: Bearer`**. **Every scan, scan-job and idea route needs one**, and `Legoin-Device-Id` goes. Scans made under a device id **move once** to the account that logs in from that device. **Photos and previews stay plain URLs** that need no header. The contract's new [Accounts](../api/contract.md#4-accounts) section has the routes and shapes.

### Accounts

- **Email.** The backend trims it and lowercases it, and keeps it that way. It's at most 254 characters, looks like `name@domain` with a dot in the domain, and has one account at most.
- **Nothing is sent to it.** With no email service, the address is never confirmed.
- **Routes:** `POST /auth/register` makes an account and logs it in, and `POST /auth/login` logs in. `POST /auth/logout` ends the session, and `GET /auth/me` reads the account.
- **Errors:**
  - registering an email that has an account answers `409 email_taken`;
  - a wrong email or password answers `401 invalid_credentials`, which doesn't say which of the two was wrong.

### Passwords

- **Rules:** 8 to 128 characters, any characters, and nothing else: no required digits or symbols.
  - That follows NIST SP 800-63B-4 on the maximum (at least 64) and on composition rules (none).
  - It doesn't follow its minimum: for a password that's the only factor, NIST asks for 15 characters. The team takes 8, NIST's minimum alongside a second factor, because people type it on phones during demos. The limit on failed log-ins and the slow hash slow down guessing. Raising the minimum later is a one-line change, and existing passwords keep working.
- **The hash:** Argon2id from `argon2-cffi`, at its defaults, RFC 9106's second recommended profile: 3 passes over 64 MiB with 4 lanes.
  - The settings live in each stored hash, so they can be raised later. A log-in then re-hashes the password with the new ones.
  - One hash took about 20 ms on an Apple M4 Pro laptop.
- **Memory.** Each hash uses 64 MiB, so the backend runs at most two at once. Other log-ins wait their turn.
- **An unknown email** is still checked against a stand-in hash, so the time taken doesn't tell whether the email has an account.
- **Never logged.** No password is logged, sent anywhere else, or stored as typed.
- **Too many failures.** After 10 failed log-ins for one email in 15 minutes, logging in to it answers `429 rate_limited` with `Retry-After`, until the oldest failure is 15 minutes old. A log-in that works clears the count. The count lives in memory, which fits ADR-0008's single process, and a restart forgets it.

### Sessions

- **The token** is `lgn_` and 32 random bytes in URL-safe base64. The client never parses it.
- **Stored hashed.** The backend keeps only its SHA-256, so a copy of the database can't be used to log in.
- **Lifetime.** It lasts 30 days from logging in, and there's no refresh. After that, the person logs in again.
- **One session per log-in,** so an account can be logged in on several devices. Logging out deletes the current session on the server, so its token stops working at once.
- **No session** on a route that needs one answers `401 unauthorized`, with `WWW-Authenticate: Bearer`. That comes before any other check: before a `404`, and before an upload is read.
- **The client** keeps the session (token, expiry and account) in `localStorage` under `legoin.session.v1`.
  - On `401 unauthorized`, it forgets the session and sends the person to log in, then back to the page they were on.
  - It sends the token only to `NEXT_PUBLIC_API_URL`.

### Which routes need a session

| Needs a session | Public |
| --- | --- |
| Every route in the contract's Scans and Ideas sections, and `GET /auth/me` | `POST /auth/register`, `POST /auth/login` and `POST /auth/logout`; photos; build previews and models; `GET /health` |

- **Another account's things.** A scan, job or idea of another account answers `404 not_found`, the same as one that doesn't exist, as another device's did.
- **No `403`.** The contract doesn't use it, since a `403` would tell the caller that the id exists.

### Scans from before accounts

`register` and `login` take an optional `deviceId`: the `Legoin-Device-Id` this browser sent before (`legoin.deviceId.v1` in `localStorage`).

- **What moves.** Scans saved under that id that no account has yet move to the account, with their photos and ideas. The move happens on a successful log-in or registration only, and then those scans never move again.
- **Then the id goes.** legoin-web sends it when it still has one, then deletes it.
- **No new proof.** The device id is the only proof, as it was for reading those scans.
- **It's temporary.** Once the deployed data has no unclaimed scans, `deviceId` and the backend's device column can be removed.

### Photos and previews

- **Photos stay as ADR-0008 made them:** an absolute URL with a random 128-bit name. A plain `<img>` loads it with no header.
- **Only the account gets the URL.** The API gives it to the scan's account and to no one else, and deleting the scan deletes the photo.
- **Anyone who has a photo's URL can open it,** just as anyone could before. Signed URLs that expire are the next step if that's ever not enough.
- **Build previews** stay public and cacheable for good ([ADR-0010](0010-library-of-checked-builds.md)).

### CORS

- **Headers:** CORS allows `Content-Type` and `Authorization`, in place of `Legoin-Device-Id`. The origins and methods don't change.
- **No cookies,** so `Access-Control-Allow-Credentials` isn't sent and a token never travels with a request the page didn't make.

### In legoin-api

- **Tables.** SQLite gets an `accounts` table and a `sessions` table. Scans get an account column, and their device id becomes optional.
- **Migration.** An existing data directory is migrated on start, in one transaction, and its scans keep their device id until they're claimed. There's no step to run by hand.
- **Dependency:** `argon2-cffi`, in the main dependencies, so CI installs it too.
- **No new environment variable.** The rate limit and the session lifetime are constants.
- **Tests** use cheap Argon2 settings, so the suite stays fast.

### In legoin-web

- **Pages:** `/[lang]/login` and `/[lang]/register`, in both languages.
- **Which pages need a log-in.** Scanning, results, ideas and history do. Visiting one without a session goes to the log-in, with the page to come back to; only a path on the web app itself is accepted as that page. The home page and the design pages stay public.
- **The header** shows the account's email and a log-out button.
- **The data layer** gets `register`, `logIn`, `logOut` and `getAccount`. The HTTP client sends the token and forgets it on `401`.
- **The mock** keeps its accounts and sessions in `localStorage`, with a SHA-256 of each password, never the password itself. It keeps each account's scans apart.
- **The end-to-end suite** makes an account before each test that needs one.

### Not in this version

- **Password reset and email checks.** Both need an email service. A person who forgets their password can't get back in; their scans stay with the old account.
- **Changing the email or the password,** and deleting an account. Scans can still be deleted from history.
- **Google sign-in.** It can come later as a second way to log in to the same accounts, with an OAuth client the team creates.
- **Limits on making accounts.** Only failed log-ins are limited.

### The answers to open questions 2 and 4

- **2. Accounts and auth:** yes, with an email and a password, required before scanning, and `Legoin-Device-Id` goes.
- **4. Whether history stays per device:** no, it belongs to the account and follows it to every device it logs in on.

This replaces ADR-0008's provisional answer to question 4 and its device-keyed scans. The rest of ADR-0008 stands, including its answer to question 3.

## Consequences

- Positive: scans are private to their account. A leaked or guessed device id no longer opens anyone's history.
- Positive: history follows the person across devices and survives clearing the browser.
- Positive: no third party, key, cost or new service, and no new environment variable. The backend stays one process with one data folder.
- Positive: logging out really ends a session, since sessions are rows the backend deletes.
- Positive: scans made before accounts aren't lost. The first log-in from that device moves them.
- Negative: the backend now keeps password hashes, which raises the stakes for its data directory. The hashes are slow to crack, but people reuse passwords.
- Negative: an 8-character minimum is below the 15 that NIST SP 800-63B-4 asks for when a password is the only factor.
- Negative: with no email service, there's no reset. A forgotten password locks the person out of their scans.
- Negative: emails aren't confirmed, so someone can register with another person's address. Registering also tells whether an email already has an account (`409 email_taken`).
- Negative: the token is in `localStorage`, where any script running on the page can read it. React escapes what it renders and the app loads no third-party scripts; a Content Security Policy would harden it further.
- Negative: the failed log-in limit is per email and in memory. Someone can lock an email out for 15 minutes by guessing wrong ten times, and a restart resets every count.
- Negative: every demo needs an account. Make one before presenting.
- Negative: the change breaks the old web app. legoin-api and legoin-web have to be deployed together; an old web build gets `401` everywhere.
- Negative: passwords and tokens cross the network, so both hosts must serve HTTPS. Vercel does, and the API's host must too.

## Pros and Cons of the Options

### An email and a password

- Pro: the team's choice. It needs nothing outside the app: no key, no console and no email service.
- Con: passwords to protect, and no way to reset one without an email service.

### Google sign-in

- Pro: no passwords stored, emails confirmed by Google, and one tap for most people.
- Con: an OAuth client in Google Cloud Console, with its origins to keep in step with every deployment. It also needs a Google account, and Google's script on the page.

### A magic link

- Pro: no passwords, and every email confirmed.
- Con: an email service with its key and its cost. The person also has to leave the app to open their mail, which is slow in a live demo.

### Required before scanning

- Pro: one way to own a scan, so there's no guest mode to build, test and explain.
- Con: every visit, and every demo, starts with a log-in.

### Optional, with guests on the device id

- Pro: people can try the app at once.
- Con: two kinds of owner on every route, and guest scans stay as weakly protected as today.

### An opaque bearer token in `localStorage`

- Pro: works across sites with no cookie, so browsers that block third-party cookies don't matter. It's also easy to send from `fetch` and from `curl`.
- Pro: a row in SQLite, so logging out and expiry work at once.
- Con: scripts on the page can read it, which an `HttpOnly` cookie would prevent.

### An `HttpOnly` cookie

- Pro: scripts can't read it.
- Con: the API is on another site, so it's a third-party cookie. Safari blocks those, and Chrome can too. It would need `SameSite=None; Secure` and a defence against cross-site requests, or the API moved under the web app's own domain.

### A signed JWT

- Pro: the backend could check it without a database read.
- Con: logging out can't end it before it expires unless the backend keeps a list anyway. One process with SQLite has no use for stateless tokens.

### Argon2id

- Pro: the hash RFC 9106 and OWASP recommend first. It's memory-hard, so it's costly to crack on GPUs, and its settings live in each hash.
- Con: a native dependency (`argon2-cffi`, with wheels for every platform the team uses), and 64 MiB per hash.

### bcrypt

- Pro: long proven, and widely used.
- Con: not memory-hard, and it only reads the first 72 bytes of a password.

### scrypt or PBKDF2 from the standard library

- Pro: no new dependency.
- Con: settings to choose and store by hand. PBKDF2 isn't memory-hard either.

### Unguessable photo URLs

- Pro: no change, and they work in a plain `<img>`, in the photo morph and in any cache.
- Con: anyone the URL reaches can open the photo until the scan is deleted.

### Signed photo URLs that expire

- Pro: a leaked URL stops working.
- Con: the URL changes on every read, which defeats caching and the photo morph between pages. It also needs a key to sign with, which is a new secret.

### Photos fetched with the token

- Pro: photos need a session like everything else.
- Con: every image has to go through `fetch` and a `blob:` URL, which changes every screen that shows a photo.

### Moving device scans once

- Pro: nothing made before accounts is lost, for a few lines of code.
- Con: a field and a column that only matter during the move.

### Leaving device scans behind

- Pro: nothing to build.
- Con: every scan made before accounts becomes unreachable, though it stays on disk.

## References

- [API contract](../api/contract.md), its new Accounts section and open questions 2 and 4
- [ADR-0008](0008-backend-stack.md): the backend stack, device-keyed scans, photo URLs and its answer to question 3
- [ADR-0010](0010-library-of-checked-builds.md): public, versioned build previews
- [NIST SP 800-63B-4](https://pages.nist.gov/800-63-4/sp800-63b.html), section 3.1.1.2, "Password Verifiers"
- [RFC 9106](https://www.rfc-editor.org/rfc/rfc9106), Argon2, and the [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [`argon2-cffi`](https://argon2-cffi.readthedocs.io/)
- legoin-web `src/lib/api/device-id.ts`, the device id this replaces
