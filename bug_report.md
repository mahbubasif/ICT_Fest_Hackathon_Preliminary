# Bug Report — CoWork API

This document lists the bugs found in the CoWork booking API. Each entry includes the symptom, the affected file, the root cause, and the fix.

---

## Bug 1 — Access Tokens Last Too Long

**File:** `app/auth.py`

### Symptom

Access tokens are supposed to expire after exactly 900 seconds, but generated access tokens remained valid for about 15 hours.

### Bug

`create_access_token()` multiplied `ACCESS_TOKEN_EXPIRE_MINUTES` by 60 and then passed the result as minutes:

```python
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
```

That made a 15-minute setting become 900 minutes.

### Fix

Use the configured value directly as minutes:

```python
lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
```

---

## Bug 2 — Logout Does Not Revoke the Token

**File:** `app/auth.py`

### Symptom

After calling `POST /auth/logout`, the same access token could still be used on authenticated endpoints.

### Bug

The logout path stored the token's `jti`, but token validation checked the user's `sub` against the revoked-token set.

### Fix

Token validation now checks `payload["jti"]`, matching the value recorded during logout.

---

## Bug 3 — Refresh Tokens Can Be Reused

**File:** `app/routers/auth.py`

### Symptom

The same refresh token could be submitted to `POST /auth/refresh` multiple times, returning new access and refresh tokens each time.

### Bug

The refresh endpoint decoded and accepted valid refresh tokens but never marked a refresh token as consumed.

### Fix

The endpoint now records used refresh-token `jti` values and rejects reuse with `401 UNAUTHORIZED`.

---

## Bug 4 — Duplicate Username Registration Returns Success

**File:** `app/routers/auth.py`

### Symptom

Registering the same username in the same organization returned the existing user instead of failing.

### Bug

The register endpoint returned the existing user object when it found a duplicate username.

### Fix

Duplicate usernames now raise:

```json
{ "code": "USERNAME_TAKEN" }
```

with HTTP status `409`, as required by the API contract.

---

## Bug 5 — Offset Datetimes Are Not Converted to UTC

**File:** `app/timeutils.py`

### Symptom

A datetime like `2026-07-10T12:00:00+06:00` was stored as `2026-07-10 12:00:00` UTC instead of `2026-07-10 06:00:00` UTC.

### Bug

`parse_input_datetime()` stripped timezone information with `replace(tzinfo=None)`, which removed the offset without converting the actual instant.

### Fix

Offset-aware datetimes are now converted to UTC first, then stored as naive UTC:

```python
dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
```

---

## Bug 6 — Booking Creation Allows a Past Start Time (Grace Window)

**File:** `app/routers/bookings.py`

### Symptom

A booking could be created with a `start_time` up to 5 minutes in the past. The contract requires `start_time` to be strictly in the future at request time, with no grace window.

### Bug

```python
if start <= now - timedelta(seconds=300):
    raise AppError(400, "INVALID_BOOKING_WINDOW", "start_time must be in the future")
```

Subtracting 300 seconds from `now` before comparing let any `start_time` within the last 5 minutes slip past the check.

### Fix

```python
if start <= now:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "start_time must be in the future")
```

---

## Bug 7 — Zero-Hour and Negative-Duration Bookings Are Accepted

**File:** `app/routers/bookings.py`

### Symptom

Sending an `end_time` equal to or earlier than `start_time` (a 0-hour or negative whole-hour span) was accepted, producing a booking with `price_cents` of zero or negative, instead of failing with `400 INVALID_BOOKING_WINDOW`.

### Bug

```python
duration_hours = int(duration_hours)
if duration_hours > MAX_DURATION_HOURS:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
```

Only the upper bound (`MAX_DURATION_HOURS = 8`) was enforced. A duration of `0` or any negative whole number of hours passed straight through, since nothing checked it against `MIN_DURATION_HOURS`.

### Fix

```python
duration_hours = int(duration_hours)
if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:
    raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
```

Enforcing the lower bound (`MIN_DURATION_HOURS = 1`) also guarantees `end_time` is strictly after `start_time`, since any duration under one whole hour is now rejected.

---

## Bug 8 — Back-to-Back Bookings Are Rejected as Conflicts

**File:** `app/routers/bookings.py`

### Symptom

Booking a room for `11:00–12:00` immediately after an existing `10:00–11:00` booking on the same room returned `409 ROOM_CONFLICT`, even though back-to-back bookings are explicitly allowed.

### Bug

```python
def _has_conflict(db: Session, room_id: int, start: datetime, end: datetime) -> bool:
    ...
    for b in existing:
        if b.start_time <= end and start <= b.end_time:
            return True
    return False
```

The overlap test used `<=` on both sides, so a new booking starting exactly when an existing one ends (or vice versa) was flagged as overlapping. The contract defines overlap with strict inequalities: `existing.start < new.end AND new.start < existing.end`.

### Fix

```python
    for b in existing:
        if b.start_time < end and start < b.end_time:
            return True
    return False
```

---

## Bug 9 — Room Conflict and Quota Checks Are Not Concurrency-Safe

**File:** `app/routers/bookings.py`

### Symptom

Firing multiple `POST /bookings` requests at the same time for the same room/slot (or from a member near their quota) could let more than one booking through — double-booking a room or exceeding the 3-booking quota — instead of all but one failing with `409 ROOM_CONFLICT` / `409 QUOTA_EXCEEDED`.

### Bug

```python
if _has_conflict(db, room.id, start, end):
    raise AppError(409, "ROOM_CONFLICT", "Room already booked for this interval")

_check_quota(db, user.id, now, start)

price_cents = room.hourly_rate_cents * duration_hours
booking = Booking(...)
db.add(booking)
db.commit()
```

The conflict check, quota check, and insert were three separate, unsynchronized steps. Concurrent requests could all read the same "no conflict / under quota" state before any of them committed a row, so all of them proceeded.

### Fix

A module-level lock now wraps the whole check-then-insert sequence so only one booking creation executes it at a time:

```python
_booking_creation_lock = threading.Lock()
...
with _booking_creation_lock:
    if _has_conflict(db, room.id, start, end):
        raise AppError(409, "ROOM_CONFLICT", "Room already booked for this interval")

    _check_quota(db, user.id, now, start)

    price_cents = room.hourly_rate_cents * duration_hours
    booking = Booking(...)
    db.add(booking)
    db.commit()
    db.refresh(booking)
```

Verified with 8 concurrent requests for the same slot (exactly 1 succeeded, 7 got `ROOM_CONFLICT`) and 6 concurrent requests against a 3-booking quota (exactly 3 succeeded, 3 got `QUOTA_EXCEEDED`).

---

## Bug 10 — Reference Code Generation Is Not Thread-Safe

**File:** `app/services/reference.py`, `app/models.py`

### Symptom

Under concurrent booking creation, two bookings could be issued the same `reference_code`.

### Bug

```python
def next_reference_code() -> str:
    current = _counter["value"]
    _format_pause()
    _counter["value"] = current + 1
    return f"CW-{current:06d}"
```

The counter was read, then (after a simulated delay) written back, with no synchronization. Two threads could both read the same `current` value before either wrote the increment, producing duplicate codes. There was also no database-level uniqueness constraint to catch it as a backstop.

### Fix

```python
_lock = threading.Lock()
...
def next_reference_code() -> str:
    with _lock:
        current = _counter["value"]
        _format_pause()
        _counter["value"] = current + 1
    return f"CW-{current:06d}"
```

`app/models.py` also adds `unique=True` to `Booking.reference_code` as a database-level safety net, matching the pattern already used on `Organization.name`:

```python
reference_code = Column(String, nullable=False, unique=True, index=True)
```

Verified with 10 concurrent bookings across 10 rooms — all 10 reference codes came back unique.

---

## Bug 11 — Refund Tiers Are Wrong at the Boundaries and Below 24 Hours

**File:** `app/routers/bookings.py`

### Symptom

Cancelling with exactly (or just over) 48 hours' notice paid out 50% instead of 100%, and cancelling with under 24 hours' notice paid out 50% instead of 0%.

### Bug

```python
notice_hours = int(notice.total_seconds() // 3600)
if notice_hours > 48:
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 50
```

Two separate problems: (1) `notice_hours` floors the notice to a whole hour and the top tier required it to be *strictly greater than* 48, so any notice in `[48h, 49h)` — including exactly 48 hours — fell through to the 50% tier instead of qualifying for 100%. (2) The final `else` branch (notice under 24 hours) paid `50` instead of `0`.

### Fix

```python
notice = booking.start_time - now
if notice >= timedelta(hours=48):
    refund_percent = 100
elif notice >= timedelta(hours=24):
    refund_percent = 50
else:
    refund_percent = 0
```

Comparing the `timedelta` directly against `timedelta(hours=48)` / `timedelta(hours=24)` removes the lossy floor-to-hour step, and the bottom tier now correctly pays 0%. Verified across five notice windows (>48h, just under 48h, just over 24h, just under 24h, ~1h) — each returns the correct tier.

---

## Bug 12 — Refund Rounding Doesn't Implement "Half-Cents Round Up"

**File:** `app/routers/bookings.py`, `app/services/refunds.py`

### Symptom

A refund that landed on a half-cent (e.g. 50% of an odd `price_cents`) could round down instead of up, and the cancel-response amount could be computed differently from the amount written to the `RefundLog`.

### Bug

```python
# app/routers/bookings.py
refund_amount_cents = round(booking.price_cents * (refund_percent / 100.0))
```
```python
# app/services/refunds.py
dollars = booking.price_cents / 100.0
refund_dollars = dollars * (percent / 100.0)
amount_cents = int(refund_dollars * 100)
```

`round()` uses banker's rounding (round-half-to-even), not "half up" — `round(50.5)` is `50`, not `51`. `int()` truncates instead of rounding at all. Both also do the math in floating point, which is imprecise for money.

### Fix

A single integer "round half up" helper in `app/services/refunds.py`:

```python
def calculate_refund_amount_cents(price_cents: int, percent: int) -> int:
    # Integer "round half up" to the nearest cent, avoiding float imprecision.
    return (price_cents * percent + 50) // 100
```

`log_refund()` now calls this instead of the float-based calculation. Verified: `price_cents=1001` at 50% now returns `501` (was rounding down before), while 100%/0% tiers remain exact.

---

## Bug 13 — Cancel Response Amount Could Differ From the Stored RefundLog Amount

**File:** `app/routers/bookings.py`, `app/services/refunds.py`

### Symptom

The `refund_amount_cents` returned by `POST /bookings/{id}/cancel` was not guaranteed to equal the `amount_cents` stored in the corresponding `RefundLog` row, since each was computed independently (see Bug 12).

### Bug

```python
refund_amount_cents = round(booking.price_cents * (refund_percent / 100.0))
log_refund(db, booking, refund_percent)   # computes its own amount internally
...
return {..., "refund_amount_cents": refund_amount_cents}
```

Two separate calculations of the same value, with no guarantee they agree.

### Fix

`log_refund()` now returns the `RefundLog` row it wrote, and the response reuses that authoritative value instead of recomputing it:

```python
refund_entry = log_refund(db, booking, refund_percent)
...
return {..., "refund_amount_cents": refund_entry.amount_cents}
```

There is now exactly one computation of the refund amount per cancellation, so the response and the stored log can never disagree. Verified by comparing the cancel response to `GET /bookings/{id}`'s `refunds[]` entry across every tier case, including under the concurrent-cancel race in Bug 14.

---

## Bug 14 — Cancellation Is Not Concurrency-Safe

**File:** `app/routers/bookings.py`

### Symptom

Firing multiple `POST /bookings/{id}/cancel` requests at the same booking at the same time could create more than one `RefundLog` entry before the booking's status actually became `"cancelled"`, instead of exactly one request succeeding and the rest getting `409 ALREADY_CANCELLED`.

### Bug

```python
if booking.status == "cancelled":
    raise AppError(409, "ALREADY_CANCELLED", "Booking already cancelled")

now = datetime.utcnow()
notice = booking.start_time - now
...
log_refund(db, booking, refund_percent)

_settlement_pause()
booking.status = "cancelled"
db.commit()
```

The already-cancelled check, the refund calculation/logging, and the status update were unsynchronized. Concurrent requests could all read `status == "confirmed"` and all proceed to call `log_refund()` before any of them committed the status change.

### Fix

A module-level lock now wraps the whole check-then-settle sequence, and the status is re-read from the database after the lock is acquired (so a request that was queued behind another cancel sees its result before deciding):

```python
_cancel_lock = threading.Lock()
...
with _cancel_lock:
    db.refresh(booking)
    if booking.status == "cancelled":
        raise AppError(409, "ALREADY_CANCELLED", "Booking already cancelled")

    now = datetime.utcnow()
    notice = booking.start_time - now
    ...
    refund_entry = log_refund(db, booking, refund_percent)

    _settlement_pause()
    booking.status = "cancelled"
    db.commit()
```

Verified with 10 concurrent cancel requests against the same booking: exactly 1 succeeded and 9 got `ALREADY_CANCELLED`, no hang, and `GET /bookings/{id}` showed exactly one `RefundLog` entry afterward.

---

## Bug 15 — CSV Export Leaks Cross-Org Bookings

**File:** `app/services/export.py`

### Symptom

An admin calling `GET /admin/export?room_id=<id>&include_all=true` with a `room_id` belonging to a *different* organization got back that other organization's bookings for the room, instead of an empty result (per Business Rule 9, cross-org resource IDs must behave as non-existent).

### Bug

```python
def fetch_bookings_raw(db: Session, room_id: int) -> list[Booking]:
    """Load every booking for a single room, ordered by id."""
    return (
        db.query(Booking)
        .filter(Booking.room_id == room_id)
        .order_by(Booking.id.asc())
        .all()
    )
...
if include_all:
    if room_id is not None:
        rows = fetch_bookings_raw(db, room_id)
    else:
        rows = _fetch_scoped(db, org_id, None, None)
else:
    rows = _fetch_scoped(db, org_id, user_id, room_id)
```

When `include_all=True` and a `room_id` was given, the code called `fetch_bookings_raw()`, which filters only by `room_id` with no join to `Room` and no `org_id` check at all — any admin could read any room's full booking history by ID, regardless of tenant.

### Fix

Route the `include_all` + `room_id` case through the already-correct, already org-scoped `_fetch_scoped()` helper (the same one the non-`include_all` path already used), instead of the unscoped raw fetch:

```python
if include_all:
    rows = _fetch_scoped(db, org_id, None, room_id)
else:
    rows = _fetch_scoped(db, org_id, user_id, room_id)
```

`_fetch_scoped` always joins `Room` and filters on `Room.org_id == org_id` before optionally narrowing by `room_id`, so a foreign `room_id` now yields zero rows. Verified: an Org B admin exporting Org A's `room_id` with `include_all=true` gets a CSV with only the header row (Org A's reference codes never appear), while Org A's own admin exporting the same room with `include_all=true` still correctly sees every member's bookings on it.

---

## Bug 16 — Rate Limiting Is Not Concurrency-Safe

**File:** `app/services/ratelimit.py`

### Symptom

Firing many concurrent `POST /bookings` requests from the same user could let more than 20 through in a rolling 60-second window instead of capping at 20 and returning `429 RATE_LIMITED` for the rest.

### Bug

```python
def record_and_check(user_id: int) -> None:
    now = time.time()
    bucket = _buckets.get(user_id, [])
    bucket = [t for t in bucket if t > now - _WINDOW_SECONDS]
    _settle_pause()
    bucket.append(now)
    _buckets[user_id] = bucket
    if len(bucket) > _MAX_REQUESTS:
        raise AppError(429, "RATE_LIMITED", "Too many booking requests")
```

Read-trim-append-write on the shared `_buckets[user_id]` list was unsynchronized. Concurrent requests could each read the same pre-append bucket, so their appends raced and the last writer's `_buckets[user_id] = bucket` silently dropped the others' entries — each request evaluated `len(bucket)` against an undercounted list.

### Fix

```python
_lock = threading.Lock()
...
def record_and_check(user_id: int) -> None:
    now = time.time()
    with _lock:
        bucket = _buckets.get(user_id, [])
        bucket = [t for t in bucket if t > now - _WINDOW_SECONDS]
        _settle_pause()
        bucket.append(now)
        _buckets[user_id] = bucket
        if len(bucket) > _MAX_REQUESTS:
            raise AppError(429, "RATE_LIMITED", "Too many booking requests")
```

Verified with 30 concurrent booking requests from one user (across 30 distinct rooms/times so only the rate limiter could gate them): exactly 20 succeeded and 10 got `RATE_LIMITED`, with no hang.

---

## Bug 17 — Inconsistent Lock Ordering Between Notifications Can Deadlock

**File:** `app/services/notifications.py`

### Symptom

A booking creation and a booking cancellation happening at the same time could hang the service indefinitely, violating the liveness rule that no combination of concurrent valid requests may hang it.

### Bug

```python
def notify_created(booking) -> None:
    with _email_lock:
        _send_email("created", booking)
        with _audit_lock:
            _write_audit("created", booking)


def notify_cancelled(booking) -> None:
    with _audit_lock:
        _write_audit("cancelled", booking)
        with _email_lock:
            _send_email("cancelled", booking)
```

`notify_created()` acquired `_email_lock` then `_audit_lock`; `notify_cancelled()` acquired them in the reverse order. If a create and a cancel ran concurrently, one could hold `_email_lock` while waiting for `_audit_lock` at the same moment the other held `_audit_lock` while waiting for `_email_lock` — a classic ABBA deadlock, hanging both requests (and eventually the server's whole thread pool) forever.

### Fix

Made `notify_cancelled()` acquire the locks in the same order as `notify_created()` (email, then audit), removing the possibility of a lock-ordering cycle:

```python
def notify_cancelled(booking) -> None:
    with _email_lock:
        _send_email("cancelled", booking)
        with _audit_lock:
            _write_audit("cancelled", booking)
```

Verified by firing 8 booking creations and 8 cancellations concurrently at the same room: all 16 completed successfully in well under the timeout, with no hang.

---

## Bug 18 — `GET /bookings` Sorted Newest-First Instead of Oldest-First

**File:** `app/routers/bookings.py` (line 137, `list_bookings`)

### Symptom

`GET /bookings` returned the caller's bookings sorted by **descending** `start_time`.
The contract requires **ascending** `start_time` (earliest first), ties broken by
ascending `id`. Because pagination slices this ordering, page contents were wrong
too — page 1 returned the most recent bookings instead of the earliest.

### Bug

```python
base.order_by(Booking.start_time.desc(), Booking.id.asc())
```

### Fix

```python
base.order_by(Booking.start_time.asc(), Booking.id.asc())
```

---

## Bug 19 — `GET /bookings` Pagination Skipped the First Page

**File:** `app/routers/bookings.py` (line 138, `list_bookings`)

### Symptom

Page N with limit L must return items `[(N−1)·L, N·L)`. The offset used
`page * limit`, skipping one full page: page 1 skipped the first `limit`
bookings, so a user with exactly `limit` bookings received an **empty** page 1
even though `total` was non-zero. Every page returned the next page's contents.

### Bug

```python
.offset(page * limit)
```

### Fix

```python
.offset((page - 1) * limit)
```

---

## Bug 20 — `GET /bookings` Ignored the Requested `limit`

**File:** `app/routers/bookings.py` (line 139, `list_bookings`)

### Symptom

The endpoint validated and echoed a `limit` (1–100) and used it to compute the
offset, but hardcoded the page size to 10 rows. With `limit > 10`, bookings
between index 10 and `limit` were returned by no page (skips); with `limit < 10`,
consecutive pages overlapped (repeats) — both violating "sequential pages never
skip or repeat items."

### Bug

```python
.offset((page - 1) * limit)
.limit(10)
```

### Fix

```python
.offset((page - 1) * limit)
.limit(limit)
```

---

## Bug 21 — Members Could Read Other Members' Bookings

**File:** `app/routers/bookings.py` (lines 164–165, `get_booking`)

### Symptom

`GET /bookings/{id}` scoped the lookup by organization but not by ownership, so
any member could read another member's booking in the same org. Booking ids are
sequential, making every booking enumerable. The contract requires another
member's booking id to return `404 BOOKING_NOT_FOUND`; only admins may read any
booking in their org. (`cancel_booking` already enforced this; the read path did
not.)

### Bug

```python
if booking is None:
    raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")

response = serialize_booking(booking)
```

### Fix

```python
if booking is None:
    raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
if user.role != "admin" and booking.user_id != user.id:
    raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")

response = serialize_booking(booking)
```

---

## Bug 22 — `GET /bookings/{id}` Returned `created_at` as `start_time`

**File:** `app/routers/bookings.py` (`get_booking`)

### Symptom

The single-booking detail endpoint overwrote the correctly-serialized
`start_time` with the booking's `created_at`, so the response reported the
record-creation time instead of the reserved slot's start. This also disagreed
with the `start_time` returned by `POST /bookings` and `GET /bookings` (list) for
the same booking.

### Bug

```python
response = serialize_booking(booking)
response["start_time"] = iso_utc(booking.created_at)   # clobbers the real start_time
response["refunds"] = [ ... ]
```

### Fix

```python
response = serialize_booking(booking)
response["refunds"] = [ ... ]
```

---

## Bug 23 — Creating a Booking Left the Usage-Report Cache Stale

**File:** `app/routers/bookings.py` (line 122, `create_booking`)

### Symptom

`GET /admin/usage-report` caches results per `(org, from, to)`. `create_booking`
invalidated the availability cache but **not** the report cache, so a usage
report pulled before a new booking kept returning stale (pre-booking) counts and
revenue. Rule §12 requires the report to reflect the current state immediately.

### Bug

```python
stats.record_create(room.id, price_cents)
cache.invalidate_availability(room.id, start.date().isoformat())
notifications.notify_created(booking)
```

### Fix

```python
stats.record_create(room.id, price_cents)
cache.invalidate_availability(room.id, start.date().isoformat())
cache.invalidate_report(user.org_id)
notifications.notify_created(booking)
```

---

## Bug 24 — Cancelling a Booking Left the Availability Cache Stale

**File:** `app/routers/bookings.py` (line 220, `cancel_booking`)

### Symptom

`GET /rooms/{id}/availability` caches busy intervals per `(room, date)`.
`cancel_booking` invalidated the report cache but **not** the availability cache,
so after a cancellation the room's availability kept showing the cancelled
booking as a busy interval. Rule §13 requires availability to reflect the current
state immediately.

### Bug

```python
stats.record_cancel(booking.room_id, booking.price_cents)
cache.invalidate_report(user.org_id)
notifications.notify_cancelled(booking)
```

### Fix

```python
stats.record_cancel(booking.room_id, booking.price_cents)
cache.invalidate_report(user.org_id)
cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())
notifications.notify_cancelled(booking)
```

---

## Bug 25 — Creating a Room Left the Usage-Report Cache Stale

**File:** `app/routers/rooms.py` (line 57, `create_room`)

### Symptom

The usage report must list every room in the org, "including rooms with zero
bookings" (§12). A report cached before a new room was created omitted that room
until some other activity happened to invalidate the cache, because `create_room`
did not invalidate the report cache.

### Bug

```python
db.add(room)
db.commit()
db.refresh(room)
return _serialize_room(room)
```

### Fix

```python
db.add(room)
db.commit()
db.refresh(room)
cache.invalidate_report(admin.org_id)
return _serialize_room(room)
```

---

## Bug 26 — Room Stats Counters Were Not Concurrency-Safe

**File:** `app/services/stats.py` (lines 10, 18, 26)

### Symptom

`GET /rooms/{id}/stats` must always equal the values derivable from the bookings,
including after bursts of concurrent activity (§14). `record_create` and
`record_cancel` performed a non-atomic read-modify-write (read counters → pause →
write), so concurrent bookings for the same room lost updates and the count and
revenue drifted below their true values.

### Bug

```python
def record_create(room_id: int, price_cents: int) -> None:
    current = _stats.get(room_id, {"count": 0, "revenue": 0})
    count, revenue = current["count"], current["revenue"]
    _aggregate_pause()
    _stats[room_id] = {"count": count + 1, "revenue": revenue + price_cents}
```

### Fix

Serialize the read-modify-write with a module-level lock (applied to both
`record_create` and `record_cancel`):

```python
_lock = threading.Lock()

def record_create(room_id: int, price_cents: int) -> None:
    with _lock:
        current = _stats.get(room_id, {"count": 0, "revenue": 0})
        count, revenue = current["count"], current["revenue"]
        _aggregate_pause()
        _stats[room_id] = {"count": count + 1, "revenue": revenue + price_cents}
```
