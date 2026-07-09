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
