# Bug Report — CoWork API

This document lists the high-impact bugs fixed in the CoWork booking API. Each entry includes the symptom, the affected file, the root cause, and the fix.

---

## Bug 1 — Access Tokens Last Too Long

**Difficulty:** Easy  
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

**Difficulty:** Easy  
**File:** `app/auth.py`

### Symptom

After calling `POST /auth/logout`, the same access token could still be used on authenticated endpoints.

### Bug

The logout path stored the token's `jti`, but token validation checked the user's `sub` against the revoked-token set.

### Fix

Token validation now checks `payload["jti"]`, matching the value recorded during logout.

---

## Bug 3 — Refresh Tokens Can Be Reused

**Difficulty:** Medium  
**File:** `app/routers/auth.py`

### Symptom

The same refresh token could be submitted to `POST /auth/refresh` multiple times, returning new access and refresh tokens each time.

### Bug

The refresh endpoint decoded and accepted valid refresh tokens but never marked a refresh token as consumed.

### Fix

The endpoint now records used refresh-token `jti` values and rejects reuse with `401 UNAUTHORIZED`.

---

## Bug 4 — Duplicate Username Registration Returns Success

**Difficulty:** Easy  
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

**Difficulty:** Medium  
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

**Difficulty:** Easy
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

**Difficulty:** Easy
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

**Difficulty:** Medium
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

**Difficulty:** Hard
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

**Difficulty:** Hard
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
