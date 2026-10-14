# DECISIONS.md

1. **Indexes.** `employees`: unique `{emp_code}` (identity, duplicate-create races), `{department, emp_code}` (filtered list, already sorted), `{joined_on}` (headcount match in the department summary). `attendance_logs`: unique `{emp_code, date}` (one record per day, punch-in race, PATCH lookup, monthly/trend matches), `{date: -1, emp_code: 1}` (list default sort, month-range analytics), `{emp_code, punch_in: -1}` (punch-out's "latest punch-in" lookup). Considered and rejected: an index on `late_minutes` — the leaderboard already narrows by month via the date index, and it would tax every write for little gain.

2. **Punch-in race.** Both requests compute the same `(emp_code, date)` and both call `insert_one` — there is no pre-check deciding anything. The unique index lets exactly one insert through; the other raises `DuplicateKeyError`, which I map to 409.

3. **Ties.** Every employee whose `$rank` (competition ranking) is ≤ `limit` is returned, so a tie at the cutoff can push the response past `limit` rows; tied rows order by `emp_code`. That matches "rank ≤ limit", not "first limit rows".

4. **Headcount.** The summary aggregates **from `employees`** (`joined_on` ≤ month end) and left-`$lookup`s the logs, so a zero-log employee keeps an empty array and still counts.

5. **At 100x** I'd stop aggregating raw punches on demand: maintain daily/monthly rollups (`$merge` on a schedule or incremented at punch time), and partition/shard `attendance_logs` by month so analytics touch only their slice.
