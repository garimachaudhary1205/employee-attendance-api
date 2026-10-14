# Employee Attendance & Analytics API

My submission for the HROne engineering assignment. FastAPI with PyMongo, and all the
application code lives in `app/main.py` as asked.

## Running it

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export MONGO_URI="mongodb://localhost:27017"   # or put both in a local .env
export MONGO_DB="attendance_db"
uvicorn app.main:app --port 8000
```

Indexes are created at startup and the calls are idempotent, so restarting is safe.
`GET /health` answers 200 once MongoDB responds to a ping and 503 before that. A local
`.env` is read for convenience but real environment variables always take priority.

`python sample_seed.py` loads the nine sample documents from `sample_data/`.

## Things worth knowing

* Instants are stored as timezone-aware UTC BSON dates and converted to and from epoch
  milliseconds at the API boundary. They are truncated to whole seconds before any
  calculation (R1). The epoch-ms fields use strict int validation, so floats, strings
  and seconds-looking values all get a 422.
* Every reported number is rounded with `Decimal` and `ROUND_HALF_UP`, because both
  Python's `round()` and Mongo's `$round` round half to even. This actually matters:
  4h07m30s is 4.125 hours, which must become 4.13 and therefore not a half day.
* Punch-in does a plain insert and lets the unique `(emp_code, date)` index decide the
  race, so parallel duplicates give exactly one 201. Punch-out closes the record with
  `find_one_and_update` on `punch_out: null`, one 200 and the rest 409. PATCH puts the
  values it read (status, punch times, history length) into the update filter, so a
  concurrent correction cannot silently drop a history entry; the loser gets a 409 and
  can retry.
* All four analytics are aggregation pipelines. The leaderboard uses `$setWindowFields`
  with `$rank` for competition ranking and applies `limit` after ranking. The trend uses
  `$densify` for the gap filling, with the two range endpoints injected through
  `$unionWith` so the series is complete even when the range has no records at all.
  Analytics read the stored derived values and never recompute them from punch times.
* `/admin/explain/{endpoint}` builds the exact query or pipeline the real endpoint runs,
  same filters, same sort, same pagination, same index hints, and returns the raw
  `executionStats` output.

## How I tested

Local MongoDB 7 in Docker. I kept my test scripts out of the repo, but they did four
things:

1. Hammered punch-in, punch-out and PATCH with parallel threads to confirm the
   exactly-one-winner behaviour and that no history entries get lost.
2. Seeded a synthetic month with the awkward cases the rules hint at: a 22:00 to 06:00
   shift punching out past midnight, a mid-month joiner, employees with no logs at all,
   weekend records, exact ties at the leaderboard cutoff, a log whose employee was
   deleted, and legacy documents missing `half_day` and `history`.
3. Recomputed all four analytics in plain Python from the seeded documents and diffed
   that against the API responses.
4. Loaded about 100k records and checked that every `/admin/explain` case shows an
   IXSCAN with no COLLSCAN, and that the slowest endpoint stays around 20ms on my
   laptop.

Nothing from the contract is missing.
