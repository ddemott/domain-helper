# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file Python CLI (`domain-helper.py`) that generates candidate domain names
(prefix + word + suffix combos across TLDs) and checks availability via the
API Ninjas Domain API, multithreaded with rate limiting, caching, checkpoint/resume,
and optional WHOIS expiry checks. No package structure — everything lives in
`domain-helper.py` and is exercised by `test_domain_helper.py`.

## Commands

Install deps:
```sh
pip install -r requirements.txt
```

Run:
```sh
python domain-helper.py
python domain-helper.py --dry-run          # preview candidates, no API calls
python domain-helper.py --help             # all CLI flags
```

Run tests (89 tests, unittest-based):
```sh
python -m unittest test_domain_helper -v
python -m unittest test_domain_helper.TestDomainChecker -v      # one class
python -m unittest test_domain_helper.TestDomainChecker.test_check_available -v  # one test
```

No lint/format tooling configured in this repo.

## Architecture

Pipeline through cooperating classes, all in `domain-helper.py`:

- **`Config`** — loads `config.yaml`, applies CLI-flag overrides on top, falls back to
  built-in defaults for anything missing. `__getattr__` exposes config keys as attributes.
- **`CandidateGenerator`** — builds the prefix×word×suffix cartesian product per TLD,
  then filters by max length / regex pattern / exclude list, then dedupes.
- **`DomainCache`** (`domain_cache.json`) — persists prior availability results for
  `cache_days` so repeat runs skip API calls for already-known domains
  (`filter_cached` removes cached candidates before hitting the network).
- **`RateLimiter`** — thread-safe global lock enforcing `min_interval` between API calls
  regardless of how many worker threads are checking concurrently.
- **`DomainChecker`** — does the actual API Ninjas HTTP call per domain, with retry +
  exponential backoff on 429/5xx, and structured 5Ws (Who/What/Where/Why/When) error
  logging on failure.
- **`CheckpointManager`** (`checkpoint.json`) — saves progress mid-run so an interrupted
  run can resume; cleared on successful completion.
- **`WhoisChecker`** — for taken domains, optionally checks WHOIS expiry and flags ones
  expiring within `whois_expiry_days`.
- **`Notifier`** — desktop notification (falls back to terminal bell) when `--notify` is set.
- **`DomainHelper`** — top-level orchestrator: for each TLD, generates candidates, filters
  cached ones, dispatches remaining checks across a thread pool (`max_workers`), prints a
  progress bar with ETA, writes per-TLD output files and a consolidated CSV
  (`available_all_<timestamp>.csv`, sorted by domain length), runs WHOIS checks, prints a
  summary report, and notifies on completion.

Config precedence: **CLI flags > `config.yaml` > built-in defaults**.

API key resolution (`load_api_key`): `DOMAIN_HELPER_API_KEY` env var takes priority over
`api_key.txt` (or whatever `api_key_file` points to).

## Files that are generated/local, not source

`api_key.txt`, `checkpoint.json`, `domain_cache.json`, `available_*.txt`,
`available_all_*.csv` are runtime artifacts, not code to edit — they're read/written by
the classes above and are gitignored.

## Tests

`test_domain_helper.py` has one `unittest.TestCase` subclass per production class listed
above, each covering happy and sad paths. Every sad-path assertion checks that error
output includes the 5Ws — keep that invariant when adding new error-logging code paths.
