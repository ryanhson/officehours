# Hold (OfficeHours) security review

Every claim below (data flow, PoC, and fix) was verified against a live local
instance (`python app.py`, Python 3.13.7, freshly seeded
`data/officehours.db`) on 2026-10-03. Replace the `[hosted URL]` / commit-hash
placeholders in §6 once you deploy the patched app for submission.

## 1. Architecture sketch

```
Browser                                   Server (Flask + gunicorn, SQLite)
  │  HTTP request                           │
  ▼                                         ▼
request ───────────────────────────▶  app.py
  │                                     @app.before_request load_user()
  │   Cookie: hold_session=<token>        ├─ init_db() + seed()  (first request)
  │ ─────────────────────────────▶       ├─ token = request.cookies["hold_session"]
  │                                        └─ SELECT users JOIN sessions
  │                                              WHERE sessions.token = ?  ─┐
  │                                                      g.user  ◀──────────┘
  │                                     routes:
  │                                        /                home.html   (public board)
  │                                        /login /register  create_session() → cookie
  │                                        /logout          DELETE FROM sessions
  │                                        /slots/new /slots  TA posts (role gate)
  │                                        /slots/<id>/book   student holds a slot
  │                                        /me               mine.html / ta_schedule.html
  │                                        /bookings/<id>     booking.html (+ private note)
  │                                        /bookings/<id>/cancel | /transfer
  │                                        /handoff/<code>    device handoff (patched)
  │  Set-Cookie: hold_session=…           │
  │ ◀─────────────────────────────  @app.after_request persist_session_cookie()
  ▼                                        ▼
                                       db.py  → data/officehours.db
                                       (users, slots, bookings, sessions, handoff_codes)
```

**What runs where.** This is a server-rendered Flask app: every route handler
in `app.py` opens a SQLite connection (`db.get_db()`), runs parameterized SQL,
and renders a Jinja template to HTML. The browser only ever receives finished
HTML plus static CSS — there is no client-side JavaScript and no JSON API
besides `GET /health`. Form submissions (`POST /login`, `/slots/<id>/book`,
`/bookings/<id>/transfer`, …) come back as ordinary form posts.

**How a login becomes a cookie the browser sends back.** On a successful
`POST /login` (`app.py:118-133`) or `POST /register`, the handler calls
`create_session(user_id)` (`app.py:76-82`), which generates a 24-byte random hex
token (`secrets.token_hex(24)` → 48 hex chars), inserts `(token, user_id)` into
the `sessions` table, and stashes it on `g.session_token`. The
`@app.after_request` hook `persist_session_cookie()` (`app.py:63-77`) then writes
it to the browser as the `hold_session` cookie. On every later request,
`@app.before_request load_user()` (`app.py:39-61`) reads that cookie, JOINs
`sessions` to `users`, and sets `g.user` — or leaves it `None` if the token is
missing or unknown. `login_required` and the `role` checks read `g.user`.

**Public board vs. booking detail.** `GET /` (`app.py:108-125`) is public: it
lists every slot with its TA, time, location, topic, and a "held / Hold this /
Sign in to hold" control — but never who holds a slot or any private note.
`GET /bookings/<id>` (`app.py:281-311`) is the private view: it requires login,
and a student may only open their own booking (`booking.student_id == user.id`,
else `abort(403)`); a TA may open any. The private TA note is shown only to the
booking's own student (`booking.html` gates it on
`current_user['id'] == booking.student_id`).

**Which actions change server state.** `book` inserts a `bookings` row;
`cancel` deletes it; `transfer` updates its `student_id`; `logout` deletes the
`sessions` row and clears the cookie; TA `create_slot` inserts a `slots` row.
Everything else (`/`, `/me`, `/bookings/<id>`) is a read.

**What the device-handoff link is doing.** On "My bookings" there is a link so
you can open the same session on another browser/device. **In the original code
this link embedded the raw, long-lived `hold_session` token directly in the URL
as `?sid=…`, and `load_user()` accepted that query parameter as a full login.
That is the defect — see §2.** (The patched version replaces it with a
short-lived, single-use handoff code; see §3.)

## 2. The defect: session token exposed in a URL → account takeover

**Location:** `load_user()` `app.py:37` (original) combined with the handoff
link in `templates/mine.html` and `mine()` passing `sid=g.session_token`.
**Class:** CWE-598 (use of GET / sensitive information in a query string),
leading to CWE-384 (session fixation / hijacking). Exploitable by anyone who
sees the URL — no password required.

The original code wired the live session token into two places it must never
go. First, `mine()` handed the token to the template:

```python
return render_template("mine.html", bookings=bookings, sid=g.session_token)
```

and `templates/mine.html` rendered it into a clickable, shareable link:

```html
Device handoff:
<a href="{{ url_for('mine', sid=sid) }}">open my bookings on this browser</a>
```

Second, `load_user()` treated that same query parameter as an authentication
credential, checked **before** the cookie:

```python
token = request.args.get("sid") or request.cookies.get("hold_session")
```

So the "device handoff" URL *is* the account. The token it carries is the real
`hold_session` value: 48 hex chars, valid for **14 days** (`max_age=60*60*24*14`),
and **reusable** — not single-use. Anyone who obtains that URL is logged in as
the victim until the token expires.

Worse, URLs are the least private part of a web request. This token leaks
through:
- **Browser history** and autocomplete on any shared/lab/library machine.
- **Server and proxy access logs** (query strings are logged by default).
- **The `Referer` header** — every external link on the page (the TA-supplied
  "Zoom (link sent after booking)" location, any link a TA types into a slot,
  even a bookmarked share) sends the full `?sid=…` URL to that third-party site.

The original cookie made it worse still: it was set with `httponly=False`
(`app.py:65`) and `app.config["SESSION_COOKIE_HTTPONLY"] = False`
(`app.py:13`), so the token is also readable from `document.cookie` by any
script — but the headline defect is that the app hands the token out in a URL
by design and then accepts it back from the URL.

### Verified trigger (local instance, 2026-10-03)

1. Sign in as the seeded student `lea@campus.edu` / `campus123` and open
   **My bookings**. The page source contains:

   ```
   href="/me?sid=bcbee432…"        (48 hex chars — the live session token)
   ```

2. Copy that URL into a **different browser with no cookies at all** (simulated
   with a fresh `curl` client — no cookie jar):

   ```
   GET /me?sid=<48-hex-token>
   ```

   The server returns `200 OK` and renders **Lea's** "My bookings" page. The
   same request a second time still returns `200` — the token is durable and
   reusable, not consumed.

3. Following the booking links from that hijacked view
   (`GET /bookings/1?sid=<token>`) renders Lea's private booking detail,
   including the TA's confidential note:

   ```
   Makeup midterm window: bring ID to Evans 204.
   Release code HOLD-2291. Do not forward this note.
   ```

**Impact.** A single leaked URL (lab-machine history, a proxy log line, or a
`Referer` sent to any site linked from the page) is a complete,
password-less, 14-day account takeover. The attacker reads the victim's private
TA notes (release codes, makeup-exam logistics explicitly marked "do not
forward"), can **cancel** the victim's holds, and can **transfer** them to any
other account — all actions that change server state on the victim's behalf. If
the victim is a TA, the same URL exposes the TA's full schedule and every
student who booked.

## 3. Fix

Two changes, both in `app.py` (plus a one-line template change and a new table):

**(a) Never authenticate from the query string.** `load_user()` now reads the
token only from the `hold_session` cookie:

```python
# Authenticate only from the session cookie. The session token is never read
# from the query string: URLs leak via history, proxy/server logs, and the
# Referer header, so a token there would be a durable account credential.
token = request.cookies.get("hold_session")
```

**(b) Make device handoff a short-lived, single-use code — not the session
token.** A new `handoff_codes` table (`db.py`) holds
`(code, user_id, created_at, used_at)`. `mine()` mints a fresh code per page
view (`create_handoff_code`, TTL 10 minutes, expired/used rows pruned on
creation) and the template links to `/handoff/<code>` instead of `?sid=`. The
new route redeems a code exactly once:

```python
@app.get("/handoff/<code>")
def handoff(code):
    conn = get_db()
    row = conn.execute(
        "SELECT * FROM handoff_codes WHERE code = ? AND used_at IS NULL "
        "AND created_at >= datetime('now', ?)",
        (code, f"-{HANDOFF_TTL_MINUTES} minutes"),
    ).fetchone()
    if not row:
        conn.close()
        flash("That device link has expired or was already used. Open My bookings to get a fresh one.")
        return redirect(url_for("login"))
    # Burn the code before minting a session so it can only ever be redeemed once.
    conn.execute("UPDATE handoff_codes SET used_at = datetime('now') WHERE code = ?", (code,))
    conn.commit()
    conn.close()
    g.session_token = create_session(row["user_id"])   # mint a NEW session for this device
    flash("Signed in on this browser.")
    return redirect(url_for("mine"))
```

The URL now carries a value that is (i) **not** the session token, (ii) valid
for **10 minutes**, and (iii) **single-use** — redeeming it mints a brand-new
session cookie for the second device and immediately burns the code. A leaked
handoff URL is worthless seconds later.

**(c) Cookie hardening** (defense-in-depth). The `hold_session` cookie is now
`HttpOnly` + `SameSite=Lax`, and `Secure` when served over HTTPS
(`_is_secure_request()` also honors `X-Forwarded-Proto` behind Fly's proxy). The
Flask config defaults (`SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SAMESITE`)
were flipped from the insecure `False`/`None` to `True`/`"Lax"`. The `fly.toml`
adds `Referrer-Policy: no-referrer`, `X-Frame-Options: DENY`, and
`X-Content-Type-Options: nosniff`.

**Why normal use still works.** Logging in, booking, cancelling, transferring,
and TA slot-posting all read `g.user` from the cookie exactly as before — none
of them ever used `?sid=`. The device-handoff feature still works end to end:
"My bookings" still shows a link you can open on another browser; that browser
still lands on the signed-in "My bookings" page. The only behavioral change is
that the link is now a disposable code instead of a permanent credential.

## 4. Patch verification (2026-10-03, local instance)

- [x] Applied the §3 patch to `app.py`, `db.py`, and `templates/mine.html`;
      added the `handoff_codes` table.
- [x] **Old attack is dead:** replaying a captured 48-hex session token as
      `GET /me?sid=<token>` from a cookieless client now returns
      `302 → /login` instead of `200` + the victim's page.
- [x] **No token leak:** the "My bookings" HTML no longer contains the 48-hex
      session token anywhere (`grep 'sid=[a-f0-9]{48}'` → no match); the handoff
      link is now `/handoff/<32-char code>`.
- [x] **Handoff still works, once:** a fresh cookieless client that opens the
      `/handoff/<code>` link is redirected to `/me`, receives a new
      `hold_session` cookie, lands on "My bookings", and can read its own
      private note (`HOLD-2291`). Opening the **same** link a second time
      returns `302 → /login` ("expired or already used").
- [x] **Normal booking intact:** as `nico@campus.edu`, `POST /slots/<id>/book`
      → `302 /me` and the booking appears; `transfer` to `lea@campus.edu`
      moves it (nico's list drops to 0); `cancel` removes it; TA `ravi` posts a
      new slot (`302 /`). The cross-student IDOR guard still holds: nico opening
      lea's `/bookings/1` returns `403`.
- [x] **Cookie flags:** `Set-Cookie: hold_session=…; HttpOnly; Path=/;
      SameSite=Lax` (and `Secure` over HTTPS on the hosted instance).
- [ ] Deploy the patched app to Fly.io (Dockerfile deploy via remote builder,
      persistent volume `officehours_data` mounted at `/app/data`, real random
      `SECRET_KEY` set via `flyctl secrets set`) and re-run the checks against
      the hosted URL — fill in §6.

## 5. Submission fields

- **Hosted URL:** `[hosted URL]` (Fly.io, region iad)
- **Date/time PoC verified against the hosted instance:** `[fill after deploy]`
- **Commit hash of the patch:** `[fill after commit]` (repo: https://github.com/ryanhson/officehours)

### Optional hardening (beyond the required fix)

- Expire sessions server-side (e.g. reject `sessions` rows older than N days in
  `load_user()`), since the 14-day cookie lifetime is otherwise the only bound.
- Add a CSRF token to the state-changing POST forms (`book`, `cancel`,
  `transfer`, `logout`); `SameSite=Lax` blocks the cross-site cases but an
  explicit token is stronger. Note `cancel`/`transfer` also accept `GET`, which
  should be tightened to `POST` so a prefetched/`<img>`-embedded link can't
  mutate state.
