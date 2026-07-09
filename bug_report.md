# Bug Report - CoWork API

## Bug 1 - Access Token Lifetime is 15 Hours Instead of 15 Minutes
- **File:** `app/auth.py`, line 50
- **What the bug was:** The access token lifetime was calculated as `timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)`. Since `ACCESS_TOKEN_EXPIRE_MINUTES = 15`, this evaluates to `timedelta(minutes=900)` = 54,000 seconds = **15 hours**. The spec requires access tokens to expire in exactly **900 seconds** (15 minutes).
- **Why it caused incorrect behavior:** Tokens lived 60× longer than intended, breaking the authentication contract.
- **How it was fixed:** Removed the erroneous `* 60` multiplier:
  Before:
  ```python
  lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
  ```
  After:
  ```python
  lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
  ```

---

## Bug 2 - Logout Does Not Actually Invalidate the Token
- **File:** `app/auth.py`, line 97
- **What the bug was:** `revoke_access_token()` (line 86) stores the token's `jti` (unique token ID) in the `_revoked_tokens` set. But the validation check on line 97 compared `payload.get("sub")` (the user's ID) against that set. Since `sub` and `jti` are completely different values, they never matched - meaning every revoked token was still accepted.
- **Why it caused incorrect behavior:** Calling `POST /auth/logout` had zero effect. The token continued to work on all subsequent requests, violating Rule 8.
- **How it was fixed:** Changed the check from `"sub"` to `"jti"`:
  Before:
  ```python
  if payload.get("sub") in _revoked_tokens:
  ```
  After:
  ```python
  if payload.get("jti") in _revoked_tokens:
  ```

---

## Bug 3 - Timezone-Aware Datetimes Not Converted to UTC
- **File:** `app/timeutils.py`, line 13
- **What the bug was:** When an input datetime carried a UTC offset (e.g. `2026-07-10T10:00:00+05:00`), the code used `dt.replace(tzinfo=None)` which simply **strips** the timezone info without converting the time value. The datetime `10:00+05:00` should become `05:00 UTC`, but instead it became `10:00 UTC` - a 5-hour error.
- **Why it caused incorrect behavior:** Bookings with timezone offsets were stored at the wrong time, causing incorrect overlap detection, wrong availability results, and wrong refund notice calculations. Violates Rule 1: "Input datetimes carrying a UTC offset must be converted to UTC before storage."
- **How it was fixed:** Used `astimezone(timezone.utc)` to actually convert to UTC before stripping:
  Before:
  ```python
  dt = dt.replace(tzinfo=None)
  ```
  After:
  ```python
  dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
  ```

---

## Bug 4 - Duplicate Username Returns 200 Instead of 409 USERNAME_TAKEN
- **File:** `app/routers/auth.py`, lines 37–43
- **What the bug was:** When a user tried to register with a username that already existed in the same org, the code silently returned the existing user's info with a 200 status instead of raising an error.
- **Why it caused incorrect behavior:** Violated Rule 15 which requires `409 USERNAME_TAKEN`. Also leaked existing user data without any password verification.
- **How it was fixed:** Replaced the return with an AppError raise:
  Before:
  ```python
  if existing is not None:
      return {"user_id": existing.id, "org_id": org.id, ...}
  ```
  After:
  ```python
  if existing is not None:
      raise AppError(409, "USERNAME_TAKEN", "Username already taken")
  ```

---

## Bug 5 - Refresh Token Not Single-Use
- **File:** `app/routers/auth.py`, lines 78-87
- **What the bug was:** The `/auth/refresh` endpoint generated a new access and refresh token pair without invalidating the refresh token that was presented.
- **Why it caused incorrect behavior:** Violates Rule 8 which requires refresh tokens to be single-use. The old refresh token could be reused indefinitely to get new tokens, posing a security risk.
- **How it was fixed:** Added a check to see if the refresh token is revoked, and then added the presented refresh token to the revoked set before returning the new tokens:
  ```python
  # Added in app/auth.py:
  def is_token_revoked(jti: str) -> bool:
      return jti in _revoked_tokens
  ```
  ```python
  # Added in app/routers/auth.py in the refresh endpoint:
  if is_token_revoked(data.get("jti")):
      raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
  
  # ... (user validation) ...
  
  # Invalidate this refresh token
  revoke_access_token(data)
  ```

---

## Bug 6 - 5-Minute Grace Window on Start Time
- **File:** `app/routers/bookings.py`, line 86
- **What the bug was:** The check `if start <= now - timedelta(seconds=300):` allowed bookings to be created up to 5 minutes in the past.
- **Why it caused incorrect behavior:** Violated Rule 2, which explicitly states "start_time must be strictly in the future at request time - no grace window."
- **How it was fixed:** Removed the 5-minute subtraction:
  Before:
  ```python
  if start <= now - timedelta(seconds=300):
  ```
  After:
  ```python
  if start <= now:
  ```

---

## Bug 7 - No Minimum Duration or End > Start Check
- **File:** `app/routers/bookings.py`, line 93
- **What the bug was:** The validation only checked `duration_hours > MAX_DURATION_HOURS`, meaning bookings with negative durations (where `end_time` is before `start_time`) or 0 hours were allowed, as long as they didn't exceed 8 hours.
- **Why it caused incorrect behavior:** Violated Rule 2, which requires `end_time` to be strictly after `start_time` and a minimum duration of 1 hour.
- **How it was fixed:** Added explicit `end <= start` and minimum duration checks:
  ```python
  # Added before calculating duration:
  if end <= start:
      raise AppError(400, "INVALID_BOOKING_WINDOW", "end_time must be after start_time")
  ```
  ```python
  # Changed duration check:
  if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:
      raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
  ```

---

## Bug 8 - Back-to-Back Bookings Rejected
- **File:** `app/routers/bookings.py`, line 50
- **What the bug was:** The conflict check in `_has_conflict` used `<=` (`if b.start_time <= end and start <= b.end_time:`).
- **Why it caused incorrect behavior:** This rejected back-to-back bookings (e.g., one ends at 3 PM, another starts at 3 PM) because the start and end times were equal. Rule 3 states "Back-to-back bookings are allowed" and uses strictly less-than `<` for overlap definition.
- **How it was fixed:** Changed `<=` to `<`:
  Before:
  ```python
  if b.start_time <= end and start <= b.end_time:
  ```
  After:
  ```python
  if b.start_time < end and start < b.end_time:
  ```

---

## Bug 9 - Pagination Logic Incorrect (3 Sub-Bugs)
- **File:** `app/routers/bookings.py`, lines 137–144
- **What the bug was:** The `GET /bookings` endpoint had three issues: 1) Sorted descending instead of ascending, 2) Used `offset(page * limit)` which skipped the first page, and 3) Hardcoded `.limit(10)` instead of using the provided `limit` parameter.
- **Why it caused incorrect behavior:** Violated Rule 11 (pagination and ordering).
- **How it was fixed:**
  Before:
  ```python
  items = (
      base.order_by(Booking.start_time.desc(), Booking.id.asc())
      .offset(page * limit)
      .limit(10)
      .all()
  )
  ```
  After:
  ```python
  items = (
      base.order_by(Booking.start_time.asc(), Booking.id.asc())
      .offset((page - 1) * limit)
      .limit(limit)
      .all()
  )
  ```

---

## Bug 10 - `start_time` Overwritten with `created_at` in Detail Response
- **File:** `app/routers/bookings.py`, line 169
- **What the bug was:** In `get_booking`, the response's `start_time` field was incorrectly overwritten by the `created_at` timestamp: `response["start_time"] = iso_utc(booking.created_at)`.
- **Why it caused incorrect behavior:** Corrupted the booking response payload schema.
- **How it was fixed:** Removed the offending line completely.

---

## Bug 11 - Members Can See Other Members' Bookings
- **File:** `app/routers/bookings.py`, line 166
- **What the bug was:** In `get_booking`, any member could fetch any booking in their org, regardless of who created it.
- **Why it caused incorrect behavior:** Violated Rule 10: "Members may read and cancel only their own bookings."
- **How it was fixed:** Added an authorization check for members:
  ```python
  if user.role != "admin" and booking.user_id != user.id:
      raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
  ```

---

## Bug 12 - Refund Percentage Thresholds Incorrect
- **File:** `app/routers/bookings.py`, lines 203–209
- **What the bug was:** The `cancel_booking` endpoint used integer division to truncate notice hours, meaning e.g., 48.5 hours became 48, which failed the `> 48` check. Also, the fallback case (`< 24h` notice) returned 50% instead of 0%.
- **Why it caused incorrect behavior:** Violated Rule 6 cancellation refund policy.
- **How it was fixed:** Compared exact `timedelta` objects and corrected the 0% case:
  Before:
  ```python
  notice_hours = int(notice.total_seconds() // 3600)
  if notice_hours > 48: ... else: refund_percent = 50
  ```
  After:
  ```python
  if notice > timedelta(hours=48):
      refund_percent = 100
  elif notice >= timedelta(hours=24):
      refund_percent = 50
  else:
      refund_percent = 0
  ```

---

## Bug 13 - Availability Cache Not Invalidated on Cancel
- **File:** `app/routers/bookings.py`, line 217
- **What the bug was:** When a booking was cancelled, the cache for usage reports was invalidated (`invalidate_report`), but the availability cache was not.
- **Why it caused incorrect behavior:** Cancelled bookings still appeared as busy intervals in `GET /rooms/{id}/availability`, violating Rule 13 ("Reflects the current state immediately").
- **How it was fixed:** Added cache invalidation for availability:
  ```python
  cache.invalidate_availability(booking.room_id, booking.start_time.date().isoformat())
  ```

---

## Bug 14 - Refund Amount Rounding Incorrect (2 locations)
- **Files:** `app/services/refunds.py` lines 15–17, `app/routers/bookings.py` line 211
- **What the bug was:** Two separate, inconsistent rounding methods:
  1. `refunds.py` converted cents→dollars→back and used `int()` (truncation). Example: 50% of 1001 cents → `int(5.005 * 100)` = 500.
  2. `bookings.py` used Python's `round()` which uses banker's rounding (round half to even). Example: `round(500.5)` = 500.
  Both gave 500, but the spec says half-cents round **up** → should be 501.
- **Why it caused incorrect behavior:** Violated Rule 6: "Refund amount rounds to the nearest cent, half-cents rounding up. RefundLog amount = cancel response amount."
- **How it was fixed:** Both locations now use the same integer arithmetic formula, which correctly implements "round half up" without any floating-point issues:
  Before (refunds.py - truncation):
  ```python
  dollars = booking.price_cents / 100.0
  refund_dollars = dollars * (percent / 100.0)
  amount_cents = int(refund_dollars * 100)
  ```
  Before (bookings.py - banker's rounding):
  ```python
  refund_amount_cents = round(booking.price_cents * (refund_percent / 100.0))
  ```
  After (both files - half-up rounding via integer math):
  ```python
  amount_cents = (booking.price_cents * percent + 50) // 100
  ```

---

## Bug 15 - Deadlock in Notifications (ABBA Lock Ordering)
- **File:** `app/services/notifications.py`, lines 24–35
- **What the bug was:** Classic ABBA deadlock. `notify_created` acquired `_email_lock` then `_audit_lock`, while `notify_cancelled` acquired `_audit_lock` then `_email_lock`. When a booking creation and cancellation happened concurrently, each thread held the lock the other needed, causing the service to hang forever.
- **Why it caused incorrect behavior:** Violated Rule 16: "No combination of concurrent requests may hang the service." Under concurrent create+cancel, the API would become completely unresponsive.
- **How it was fixed:** Changed `notify_cancelled` to acquire locks in the same order as `notify_created` (email_lock first, then audit_lock):
  Before (ABBA deadlock):
  ```python
  def notify_cancelled(booking):
      with _audit_lock:
          _write_audit(...)
          with _email_lock:
              _send_email(...)
  ```
  After (consistent AB ordering):
  ```python
  def notify_cancelled(booking):
      with _email_lock:
          _send_email(...)
          with _audit_lock:
              _write_audit(...)
  ```

---

## Bug 16 - Reference Code Race Condition (Duplicate Codes)
- **File:** `app/services/reference.py`, lines 17–21
- **What the bug was:** The `next_reference_code()` function reads the counter, sleeps 120ms (`_format_pause()`), then writes the incremented value. Under concurrent requests, two threads can read the same counter value during the sleep, both increment to the same new value, and produce duplicate reference codes.
- **Why it caused incorrect behavior:** Violated Rule 7: "Every booking's reference_code is unique, including under concurrent creation."
- **How it was fixed:** Added a `threading.Lock` around the entire read-sleep-write block:
  ```python
  _counter_lock = threading.Lock()

  def next_reference_code() -> str:
      with _counter_lock:
          current = _counter["value"]
          _format_pause()
          _counter["value"] = current + 1
      return f"CW-{current:06d}"
  ```

---

## Bug 17 - Stats Race Condition (Lost Updates)
- **File:** `app/services/stats.py`, lines 15–26
- **What the bug was:** Both `record_create` and `record_cancel` used the same read→sleep(100ms)→write pattern as the reference code bug. Two concurrent creates on the same room would both read count=0, sleep, then both write count=1 - losing one increment.
- **Why it caused incorrect behavior:** Violated Rule 14: "Room stats must always be consistent with the bookings themselves, including after bursts of concurrent activity."
- **How it was fixed:** Added a `threading.Lock` around both `record_create` and `record_cancel`:
  ```python
  _stats_lock = threading.Lock()

  def record_create(room_id, price_cents):
      with _stats_lock:
          current = _stats.get(room_id, {"count": 0, "revenue": 0})
          count, revenue = current["count"], current["revenue"]
          _aggregate_pause()
          _stats[room_id] = {"count": count + 1, "revenue": revenue + price_cents}
  ```

---

## Bug 18 - Export Cross-Org Data Leak
- **File:** `app/services/export.py`, lines 48–50
- **What the bug was:** When `include_all=True` and a `room_id` was specified, the code called `fetch_bookings_raw(db, room_id)` which queries bookings by `room_id` alone - **no org_id filter**. If an admin guessed or brute-forced another org's room ID, they could export that org's booking data.
- **Why it caused incorrect behavior:** Violated Rule 9: "A user may only ever read or act on data belonging to their own organization, on every code path."
- **How it was fixed:** Replaced `fetch_bookings_raw` with `_fetch_scoped` which always joins through Room and filters by `org_id`:
  Before (no org filter):
  ```python
  rows = fetch_bookings_raw(db, room_id)
  ```
  After (org-scoped):
  ```python
  rows = _fetch_scoped(db, org_id, None, room_id)
  ```

---

## Bug 19 - Rate Limiter Race Condition
- **File:** `app/services/ratelimit.py`, lines 18–26
- **What the bug was:** The `record_and_check` function reads the user's request bucket, sleeps 100ms (`_settle_pause()`), appends the current timestamp, then checks the count. Under concurrent requests, all threads read the same bucket (e.g., 19 entries), all sleep, all append (now 20 each), and all pass the `> 20` check - allowing far more than 20 requests through.
- **Why it caused incorrect behavior:** Violated Rule 5: "POST /bookings is limited to 20 requests per rolling 60 seconds per user. Must hold under concurrent requests."
- **How it was fixed:** Added a `threading.Lock` around the entire read-sleep-append-check block:
  ```python
  _rate_lock = threading.Lock()

  def record_and_check(user_id: int) -> None:
      with _rate_lock:
          now = time.time()
          bucket = _buckets.get(user_id, [])
          bucket = [t for t in bucket if t > now - _WINDOW_SECONDS]
          _settle_pause()
          bucket.append(now)
          _buckets[user_id] = bucket
          if len(bucket) > _MAX_REQUESTS:
              raise AppError(429, "RATE_LIMITED", "Too many booking requests")
  ```

---

## Bug 20 - Usage Report Cache Not Invalidated on Booking Creation
- **File:** `app/routers/bookings.py`, line 124
- **What the bug was:** When a member creates a new booking, the code invalidated the availability cache but forgot to invalidate the admin usage report cache for their organization. 
- **Why it caused incorrect behavior:** Violated Rule 12: "Usage report... must reflect the current state immediately." If an admin cached the report, they wouldn't see newly created bookings until a cancellation eventually cleared the cache.
- **How it was fixed:** Added the missing `cache.invalidate_report` call:
  Before:
  ```python
  stats.record_create(room.id, price_cents)
  cache.invalidate_availability(room.id, start.date().isoformat())
  ```
  After:
  ```python
  stats.record_create(room.id, price_cents)
  cache.invalidate_report(user.org_id)
  cache.invalidate_availability(room.id, start.date().isoformat())
  ```

---

## Bug 21 - Usage Report Cache Not Invalidated on Room Creation
- **File:** `app/routers/rooms.py`, line 57
- **What the bug was:** When an admin creates a new room, the usage report cache was not invalidated.
- **Why it caused incorrect behavior:** Violated Rule 12: "Usage report... returns, per room in the caller's org (including rooms with zero bookings)... Must reflect the current state immediately." The newly created room (which has zero bookings) would not show up in the cached report.
- **How it was fixed:** Cleared the report cache right before returning the new room:
  Before:
  ```python
  db.add(room)
  db.commit()
  db.refresh(room)
  return _serialize_room(room)
  ```
  After:
  ```python
  db.add(room)
  db.commit()
  db.refresh(room)
  cache.invalidate_report(admin.org_id)
  return _serialize_room(room)
  ```

---

## Bug 22 - Room Double-Booking Under Concurrency
- **File:** `app/routers/bookings.py`, line 102
- **What the bug was:** The `create_booking` endpoint evaluated `_has_conflict` (which queried the DB for overlaps), then called `_pricing_warmup()` which paused for 50ms, and only then inserted the new booking. Under concurrent load, multiple threads would evaluate `_has_conflict` to `False` simultaneously, sleep, and then all insert overlapping bookings for the exact same room and time.
- **Why it caused incorrect behavior:** Violated Rule 3: "Conflict -> 409 ROOM CONFLICT. Must hold under concurrent requests."
- **How it was fixed:** Introduced a `_booking_lock = threading.Lock()` and wrapped the conflict evaluation and database insert within the lock to serialize overlapping requests across threads.

---

## Bug 23 - Quota Limit Bypassed Under Concurrency
- **File:** `app/routers/bookings.py`, line 105
- **What the bug was:** Similar to Bug 22, the `_check_quota` function read the user's current booking count, then slept for 100ms via `_quota_audit()`. Under concurrent load, a user at 2 bookings who sent 5 concurrent requests would have all 5 threads read the count as 2, pass the check, and insert 5 new bookings - ending up with 7 bookings total in a 24-hour window.
- **Why it caused incorrect behavior:** Violated Rule 4: "A member may hold at most 3 confirmed bookings... Must hold under concurrent requests."
- **How it was fixed:** The `_booking_lock` added in Bug 22 encompasses the `_check_quota` call and the subsequent `db.commit()`, preventing multiple threads from reading the same pre-insertion quota state.

---

## Bug 24 - Duplicate Refund Logs on Concurrent Cancellations
- **File:** `app/routers/bookings.py`, line 198
- **What the bug was:** The `cancel_booking` endpoint checked if `booking.status == "cancelled"`, logged the refund, slept via `_settlement_pause()`, and then set the status to "cancelled" and committed. Under concurrent cancellation requests for the exact same booking, both threads would pass the initial status check, and both would insert a new `RefundLog` row.
- **Why it caused incorrect behavior:** Violated Rule 6: "A cancelled booking has exactly one RefundLog entry... Must hold under concurrent cancel requests for the same booking."
- **How it was fixed:** Wrapped the status check, refund math, and status update within `_booking_lock`, ensuring that the second thread is blocked until the first completes. Additionally, added `db.refresh(booking)` inside the lock so the second thread sees the updated "cancelled" status and correctly throws the 409 error.

