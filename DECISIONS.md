# DECISIONS.md

1. **Indexes.** On `employees`: a unique `{emp_code}` for identity and duplicate creates,
`{department, emp_code}` so the filtered employee list comes straight off the index already
sorted, and `{joined_on}` for the headcount match in the department summary. On
`attendance_logs`: a unique `{emp_code, date}` (one record per day, the punch-in race, PATCH
lookups), `{date: -1, emp_code: 1}` (default list sort plus month ranges), and
`{emp_code, punch_in: -1}` (punch-out looks for the latest punch-in). I considered an index on
`late_minutes` for the leaderboard and dropped it: the month's date range already narrows the
scan, and the extra index would cost every write.

2. **Punch-in race.** Both requests build the same `(emp_code, date)` document and both call
`insert_one`. There is no pre-check deciding anything. The unique index lets exactly one
through; the other raises `DuplicateKeyError`, which I turn into a 409.

3. **Ties.** Everyone whose competition rank (`$rank`) is at most `limit` is returned, so a tie
at the cutoff can push the response past `limit` rows. Tied rows order by `emp_code`.

4. **Headcount.** The summary pipeline starts from `employees` (`joined_on` on or before month
end) and left-joins the logs, so a person with zero logs keeps an empty array and still counts.

5. **At 100x** I would stop aggregating raw punches per request: maintain daily and monthly
rollups with `$merge` (or increment them at punch time) and partition the logs by month, so
analytics only touch their own slice.
