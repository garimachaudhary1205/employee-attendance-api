# Employee Attendance & Analytics API

FastAPI + MongoDB implementation of the HROne assignment. All application code lives in
`app/main.py`.

## Run

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export MONGO_URI="mongodb://localhost:27017"   # or put it in a local .env
export MONGO_DB="attendance_db"
uvicorn app.main:app --port 8000
```

All indexes are created automatically at startup (idempotent). `GET /health` returns 200 once
MongoDB answers a ping, 503 otherwise. A local `.env` is read for convenience, but real
environment variables always win.

`python sample_seed.py` loads the nine sample documents from `sample_data/`.

## Notes

* Instants are stored as timezone-aware UTC BSON dates and converted to/from epoch **milliseconds**
  at the API boundary; they are truncated to whole seconds before any calculation (R1). Timestamps
  that look like seconds, floats or strings are rejected with 422 (strict int validation).
* Punch-in relies on the unique `(emp_code, date)` index for race safety (one 201, rest 409);
  punch-out closes atomically with `find_one_and_update` on `punch_out: null` (one 200, rest 409);
  PATCH uses optimistic concurrency (current punches/status/history length as the update filter),
  so concurrent corrections cannot silently lose history entries — the loser gets 409.
* All analytics are MongoDB aggregation pipelines (`$lookup`, `$setWindowFields` with `$rank` for
  competition ranking, `$densify` for the gap-filled trend). Reported numbers are rounded half-up
  (2 decimals, rates 4) using `Decimal`, since both float `round()` and Mongo's `$round` are
  round-half-to-even.
* `GET /admin/explain/{endpoint}` runs `explain` with `executionStats` on the same query/pipeline
  the endpoint builds (same filters, sort, pagination, hints). On a ~100k-record dataset every
  plan is an IXSCAN with no COLLSCAN.
* REVIEW.md lists the defects found in the starter (Part A); DECISIONS.md answers the five design
  questions (Part C).

Nothing is missing — every endpoint in the contract is implemented.
