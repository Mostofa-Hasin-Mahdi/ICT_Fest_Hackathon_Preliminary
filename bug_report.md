# Bug Report

## Bug 1: Incorrect Timezone Handling for Offset-Aware Input

* **File(s) / Line(s):** `app/timeutils.py`, lines 11-14
* **What the bug was and why it caused incorrect behavior:**
  The `parse_input_datetime` function used `dt.replace(tzinfo=None)` when an input datetime contained a UTC offset. This simply stripped the timezone information without converting the underlying time to UTC. For example, an input of `2026-07-09T18:12:04+06:00` would be incorrectly stored as `18:12:04` naive UTC instead of the correct `12:12:04` UTC, violating Business Rule 1 which states that input datetimes carrying a UTC offset must be converted to UTC before storage.
* **How it was fixed:**
  Changed `dt.replace(tzinfo=None)` to `dt.astimezone(timezone.utc).replace(tzinfo=None)`. This ensures the time is accurately shifted to UTC before the tzinfo is dropped for storage.

## Bug 2: Back-to-Back Double Booking Rejection

* **File(s) / Line(s):** `app/routers/bookings.py`, line 50
* **What the bug was and why it caused incorrect behavior:**
  In `_has_conflict`, the overlap check condition used inclusive inequality `b.start_time <= end and start <= b.end_time`. When a user attempted to book a room back-to-back right after another booking finished (for example, existing booking ends at `12:00` and new booking starts at `12:00`), `start <= b.end_time` evaluated to `12:00 <= 12:00`, which is `True`. This caused valid back-to-back bookings to be wrongly rejected with a `409 ROOM CONFLICT` error, violating Business Rule 3 (`existing.start < new.end AND new.start < existing.end. Back-to-back bookings are allowed`).
* **How it was fixed:**
  Changed `b.start_time <= end and start <= b.end_time` to `b.start_time < end and start < b.end_time`. Strict inequality ensures that back-to-back bookings where `new.start == existing.end` do not trigger false conflicts.

## Bug 3: Access Token Expiry Duration Multiplier

* **File(s) / Line(s):** `app/auth.py`, line 50
* **What the bug was and why it caused incorrect behavior:**
  The `create_access_token` function calculated the token `lifetime` as `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)`. Since `ACCESS_TOKEN_EXPIRE_MINUTES` is defined as 15 in `config.py`, this evaluated to 900 minutes instead of the intended 900 seconds (15 minutes). This violated Business Rule 8, which dictates that access tokens must expire in exactly 900 seconds.
* **How it was fixed:**
  Removed the `* 60` multiplier so the parameter matches the minutes unit, changing `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)` to `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)`.

## Bug 4: Booking Sort Order

* **File(s) / Line(s):** `app/routers/bookings.py`, line 137
* **What the bug was and why it caused incorrect behavior:**
  In `list_bookings`, the query sorted bookings in descending order of their start time (`Booking.start_time.desc()`). This violated Business Rule 11 (`question.md` Section 4), which mandates that the caller's bookings must be sorted ascending by start time (`ties by ascending id`).
* **How it was fixed:**
  Changed `Booking.start_time.desc()` to `Booking.start_time.asc()`.

## Bug 5: Pagination 1-Based Offset & Hardcoded Limit

* **File(s) / Line(s):** `app/routers/bookings.py`, lines 138-139
* **What the bug was and why it caused incorrect behavior:**
  In `list_bookings`, the SQL offset was computed as `.offset(page * limit)`. Since `page` is 1-indexed (`page=1` by default), requesting the first page computed `1 * 10 = 10`, which skipped the first 10 items (`items 1 to 10`) entirely and caused `page=1` to return `items 11 to 20` or an empty list. Furthermore, line 139 hardcoded `.limit(10)` instead of passing the dynamic `limit` parameter, causing requests with custom limits (e.g., `limit=50`) to still return at most 10 items. This violated Business Rule 11 (`Sequential pages never skip or repeat items... limit 1..100`).
* **How it was fixed:**
  Changed `.offset(page * limit)` to `.offset((page - 1) * limit)` so `page=1` correctly maps to offset `0`. Changed `.limit(10)` to `.limit(limit)` to dynamically respect the caller's requested limit.

## Bug 6: Duplicate Username Response

* **File(s) / Line(s):** `app/routers/auth.py`, lines 37-43
* **What the bug was and why it caused incorrect behavior:**
  In `register`, when a user attempted to register with an existing username inside the same organization, the code checked `if existing is not None:` and returned a dictionary containing the existing user's details with HTTP status `200/201 Created`. This violated Section 5 of `question.md` (`USERNAME TAKEN (409)`), which strictly mandates raising an application error when a username is already taken.
* **How it was fixed:**
  Changed the block to `if existing is not None: raise AppError(409, "USERNAME_TAKEN", "Username already taken")`.

## Bug 7: Cancellation Notice Refund Tiers & 48h Boundary

* **File(s) / Line(s):** `app/routers/bookings.py`, lines 200-206
* **What the bug was and why it caused incorrect behavior:**
  In `cancel_booking`, the notice calculation truncated `notice.total_seconds() // 3600` to an integer (`notice_hours`) and checked `if notice_hours > 48:`. This failed when notice was exactly 48 hours or carried fractional hours (`48.5h` truncated to `48 > 48 == False`), causing qualifying cancellations to receive `50%` instead of `100%`. Furthermore, when notice was under 24 hours, the `else:` block hardcoded `refund_percent = 50`. Both issues violated Business Rule 6 (`question.md` Section 3), which states that cancellations with `notice >= 48 hours` receive a `100%` refund, and those with `notice < 24 hours` receive a `0%` refund.
* **How it was fixed:**
  Removed the integer truncation (`notice_hours`) and used exact `timedelta` comparisons: `if notice >= timedelta(hours=48): refund_percent = 100`, followed by `elif notice >= timedelta(hours=24): refund_percent = 50`, and changed the fall-through branch to `else: refund_percent = 0`.

## Bug 8: Invalid Grace Window

* **File(s) / Line(s):** `app/routers/bookings.py`, line 86
* **What the bug was and why it caused incorrect behavior:**
  In `create_booking`, the validation check allowed bookings up to 300 seconds (5 minutes) in the past (`if start <= now - timedelta(seconds=300):`). This violated Business Rule 2 (`question.md` Section 3), which mandates that the start time must be strictly in the future at request time without any grace window.
* **How it was fixed:**
  Changed `if start <= now - timedelta(seconds=300):` to `if start <= now:`.

## Bug 9: CSV Header Mismatch

* **File(s) / Line(s):** `app/services/export.py`, lines 10-19
* **What the bug was and why it caused incorrect behavior:**
  In `export.py`, the `EXPORT_HEADER` list defined column names using underscores (`"reference_code"`, `"room_id"`, `"user_id"`, `"start_time"`, `"end_time"`, `"price_cents"`). Section 5 of `question.md` specifically mandates exact space-separated column headers (`Export CSV header (exact): id,reference code,room id,user id, start time, end time,status,price cents`). Using underscores causes automated grading checks and integration tests validating exact CSV header format to fail.
* **How it was fixed:**
  Replaced all underscored column names in `EXPORT_HEADER` with exact space-separated strings matching the specification (`"reference code"`, `"room id"`, `"user id"`, `"start time"`, `"end time"`, `"price cents"`).

## Bug 10: Broken Token Revocation (`sub` vs `jti`)

* **File(s) / Line(s):** `app/auth.py`, line 97
* **What the bug was and why it caused incorrect behavior:**
  In `get_token_payload`, the code checked whether a token was revoked by looking up the user ID (`payload.get("sub")`) inside the `_revoked_tokens` set (`if payload.get("sub") in _revoked_tokens:`). However, `revoke_access_token` correctly stores `payload["jti"]` (the unique token UUID string) inside `_revoked_tokens` upon logout. Because the check compared the user ID (`"sub"`) against a set containing token UUIDs (`"jti"`), it always evaluated to `False`. Consequently, logged-out access tokens were never rejected and could make API requests indefinitely after `POST /auth/logout`, violating Business Rule 8 (`question.md` Section 3).
* **How it was fixed:**
  Changed `if payload.get("sub") in _revoked_tokens:` to `if payload.get("jti") in _revoked_tokens:` inside `get_token_payload` (`app/auth.py`).

## Bug 11: Refresh Token Single-Use Violation

* **File(s) / Line(s):** `app/routers/auth.py`, lines 76-88
* **What the bug was and why it caused incorrect behavior:**
  In `refresh`, when a client presented a valid refresh token (`POST /auth/refresh`), the endpoint issued a new access and refresh token pair without verifying whether the presented refresh token had already been revoked, and without adding the presented refresh token (`data["jti"]`) to `_revoked_tokens`. This allowed infinite reuse of a single refresh token, directly violating Business Rule 8 (`question.md` Section 3) which mandates that refresh tokens are single use (`exchanging a refresh token returns a new pair and invalidates the presented refresh token... reuse -> 401`).
* **How it was fixed:**
  Imported `_revoked_tokens` from `..auth` into `app/routers/auth.py`, added a check `if data.get("jti") in _revoked_tokens: raise AppError(401, "UNAUTHORIZED", "Refresh token has been revoked")` upon entry, and called `revoke_access_token(data)` before returning the new token pair.

## Bug 12: Usage Report Cache Invalidation & Date Range Defaults

* **File(s) / Line(s):** `app/routers/bookings.py`, line 122; `app/routers/admin.py`, lines 20-28
* **What the bug was and why it caused incorrect behavior:**
  In `create_booking`, when a new booking was successfully placed, the code called `cache.invalidate_availability(room.id, ...)` to clear the availability cache, but omitted calling `cache.invalidate_report(user.org_id)`. As a result, subsequent queries to `GET /admin/usage-report` served stale cached data that did not reflect newly created bookings until another mutation (like a cancellation) occurred. Furthermore, in `usage_report` (`admin.py`), the `from` and `to` query parameters were marked as required (`Query(...)`), causing requests without explicit date parameters to fail with a `422 Missing required query parameter` error. Both issues violated Business Rule 9 (`question.md` Section 3), which states that the admin usage report must reflect the current state immediately upon booking changes (`cache invalidation on mutations`) and that `from` / `to` parameters should default to the last 30 days when omitted.
* **How it was fixed:**
  Added `cache.invalidate_report(user.org_id)` to `create_booking` (`app/routers/bookings.py`) immediately following `cache.invalidate_availability(...)`. Updated `usage_report` (`app/routers/admin.py`) to allow optional query parameters (`frm: str | None = Query(None, alias="from")`, `to: str | None = Query(None)`) and added fallback logic defaulting `from` to 30 days in the past (`(datetime.utcnow().date() - timedelta(days=30)).isoformat()`) and `to` to the current UTC date (`datetime.utcnow().date().isoformat()`) when either parameter is omitted.

## Bug 13: Cross-Tenant Data Leak in CSV Export

* **File(s) / Line(s):** `app/services/export.py`, line 50
* **What the bug was and why it caused incorrect behavior:**
  In `generate_export`, when an admin requested a CSV export with `include_all=True` alongside a specific `room_id`, the function called `fetch_bookings_raw(db, room_id)`. `fetch_bookings_raw` queried `Booking.room_id == room_id` without joining `Room` or filtering by `Room.org_id == org_id`. This allowed an admin from one organization to pass the `room_id` of a room belonging to a different organization and download all of that room's bookings, violating multi-tenant isolation and Business Rule 10 (`question.md` Section 3).
* **How it was fixed:**
  Replaced `fetch_bookings_raw(db, room_id)` with `_fetch_scoped(db, org_id, None, room_id)` (`app/services/export.py`), ensuring that `Room` is joined and the `.filter(Room.org_id == org_id)` check is strictly enforced on all exported booking records.

## Bug 14: Race Condition in Reference Code Generation

* **File(s) / Line(s):** `app/services/reference.py`, lines 17-21
* **What the bug was and why it caused incorrect behavior:**
  In `next_reference_code`, the function read the counter value (`current = _counter["value"]`), invoked an artificial sleep (`_format_pause()`, `time.sleep(0.12)`), and then incremented and reassigned the counter (`_counter["value"] = current + 1`). Under concurrent load, multiple threads executing `next_reference_code` simultaneously read the exact same `current` value before any thread completed its pause and wrote the incremented value back. Consequently, concurrent booking requests received duplicate reference codes and lost counter increments, violating Business Rule 5 and concurrency consistency requirements (`question.md` Section 3).
* **How it was fixed:**
  Imported `threading`, instantiated a module-level mutex `_lock = threading.Lock()`, and wrapped the read-pause-increment cycle inside `next_reference_code` within a `with _lock:` block (`app/services/reference.py`) to ensure atomic, sequential reference code issuance under concurrency.

## Bug 15: Race Condition in Rate Limiting

* **File(s) / Line(s):** `app/services/ratelimit.py`, lines 18-26
* **What the bug was and why it caused incorrect behavior:**
  In `record_and_check`, the rate limit check retrieved a user's timestamp list (`bucket = _buckets.get(user_id, [])`), trimmed expired timestamps, invoked an artificial sleep (`_settle_pause()`, `time.sleep(0.1)`), appended the new timestamp, and verified `if len(bucket) > _MAX_REQUESTS: raise AppError(429, ...)`. Under concurrent burst requests from a single user, multiple threads read the exact same `bucket` state simultaneously before any thread finished `_settle_pause()` and wrote back to `_buckets`. Each thread checked `len(bucket)` against only its own single-item local list (`len(bucket) == 1`), causing `len(bucket) > 20` to never evaluate to `True`. This allowed concurrent requests to completely bypass the `20 requests per 60 seconds` rate limit, violating Business Rule 4 and concurrency constraints (`question.md` Section 3).
* **How it was fixed:**
  Imported `threading`, instantiated a module-level mutex `_lock = threading.Lock()`, and wrapped the entire read-pause-append-check sequence inside `record_and_check` within a `with _lock:` block (`app/services/ratelimit.py`) to ensure atomic, accurate rate limit accounting across concurrent requests.
