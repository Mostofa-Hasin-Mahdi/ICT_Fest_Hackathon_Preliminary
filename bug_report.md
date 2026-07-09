# Bug Report

## Bug 1: Incorrect Timezone Handling for Offset-Aware Input

* **File(s) / Line(s):** `app/timeutils.py`, lines 11-14
* **What the bug was and why it caused incorrect behavior:**
  The `parse_input_datetime` function used `dt.replace(tzinfo=None)` when an input datetime contained a UTC offset. This simply stripped the timezone information without converting the underlying time to UTC. For example, an input of `2026-07-09T18:12:04+06:00` would be incorrectly stored as `18:12:04` naive UTC instead of the correct `12:12:04` UTC, violating Business Rule 1 which states that input datetimes carrying a UTC offset must be converted to UTC before storage.
* **How it was fixed:**
  Changed `dt.replace(tzinfo=None)` to `dt.astimezone(timezone.utc).replace(tzinfo=None)`. This ensures the time is accurately shifted to UTC before the tzinfo is dropped for storage.
