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
