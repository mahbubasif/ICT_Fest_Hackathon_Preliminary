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

## Bug 6 — `GET /bookings` Sorted Newest-First Instead of Oldest-First

**Difficulty:** Easy  
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

## Bug 7 — `GET /bookings` Pagination Skipped the First Page

**Difficulty:** Medium  
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

## Bug 8 — `GET /bookings` Ignored the Requested `limit`

**Difficulty:** Easy  
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

## Bug 9 — Members Could Read Other Members' Bookings

**Difficulty:** Medium  
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

## Bug 10 — `GET /bookings/{id}` Returned `created_at` as `start_time`

**Difficulty:** Easy  
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

## Bug 11 — Creating a Booking Left the Usage-Report Cache Stale

**Difficulty:** Medium  
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

## Bug 12 — Cancelling a Booking Left the Availability Cache Stale

**Difficulty:** Medium  
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

## Bug 13 — Creating a Room Left the Usage-Report Cache Stale

**Difficulty:** Medium  
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

## Bug 14 — Room Stats Counters Were Not Concurrency-Safe

**Difficulty:** Hard  
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
