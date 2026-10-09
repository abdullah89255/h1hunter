<!--
  README.md for h1hunter
  GitHub-flavored Markdown. Designed to render cleanly on github.com.
  Replace placeholder URLs (github.com/USERNAME/h1hunter) with your actual
  repository path before publishing.
-->

# h1hunter

> Safe-by-default reconnaissance and low-impact security check toolkit for **authorized** HackerOne bug bounty programs.

[![Python](https://img.shields.io/badge/python-3.12%2B-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Tests](https://img.shields.io/badge/tests-pytest-informational.svg)](tests/)
[![Safety](https://img.shields.io/badge/safety-regression%20suite-red.svg)](tests/integration/test_safety_regression.py)
[![Code style: ruff](https://img.shields.io/badge/code%20style-ruff-000000.svg)](https://github.com/astral-sh/ruff)

`h1hunter` is a Python CLI that helps bug bounty researchers work inside an
explicitly defined program scope. Every active operation — DNS query, HTTP
probe, security check — routes through a centralized **safety engine** that
enforces scope, destination policy, rate limits, request budgets, and a kill
switch. It does **not** exploit, fuzz, brute force, or bypass anything.

> **Authorization depends on the current program policy — not on domain
> ownership, not on technical reachability.** Only test assets you are
> explicitly permitted to test.

---

## Table of contents

- [Features](#features)
- [Safety model](#safety-model)
- [Requirements](#requirements)
- [Installation](#installation)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Commands](#commands)
- [Architecture](#architecture)
- [Testing](#testing)
- [Project layout](#project-layout)
- [Extending the toolkit](#extending-the-toolkit)
- [Known limitations](#known-limitations)
- [Responsible use](#responsible-use)
- [Contributing](#contributing)
- [License](#license)

---

## Features

**Scope enforcement**
- Wildcard, exact, URL-prefix, IP-literal, and CIDR scope entries
- Label-anchored wildcard matching that rejects lookalike hosts
- Explicit exclusions evaluated *before* inclusions
- Fail-closed on missing, empty, or unparseable scope files
- Every decision produces an auditable `ScopeDecision`

**Safety engine**
- Global token-bucket rate limiter
- Hard request budget per run
- Circuit breaker for repeated failures
- Kill switch that stops new requests immediately
- Destination policy blocking loopback, link-local, private, multicast,
  unspecified, reserved, and cloud metadata endpoints
- Metadata endpoints stay blocked even when private networks are authorized
- Append-only in-memory audit trail

**Reconnaissance**
- Pluggable passive providers (local file, newline-delimited text)
- Opt-in DNS inspection with per-record-type isolation
- Bounded HTTP probing with redirect re-validation hop by hop
- Response bodies truncated at a configured cap and never persisted
- Concurrency limited by a shared semaphore

**Checks**
- HTTP security-header inspection with contextual severity
- TLS certificate validity, expiry, and protocol inspection
- Small allowlist of well-known paths (exposure)
- Conventional API documentation location discovery
- Harmless unique-marker input reflection detection

**Reporting**
- Normalized `Finding` model with explicit confidence and verification status
- Content-addressed IDs and deduplication
- JSON, CSV, and Markdown exports (Markdown includes a HackerOne draft)
- Every export refuses to overwrite an existing file

---

## Safety model

```
                    passive file → Asset(unvalidated)
                                          │
                                          │  recorded only; never probed
                                          ▼
passive providers ────────────────────────┘
                                          │
                                          ▼
                              (explicit call) ScopeValidator.evaluate()
                                          │
                                          ▼
                              in_scope / out_of_scope / excluded

     active modules ──▶ SafetyEngine.authorize(target)
                          ├─ require_active_mode()      kill switch + active flag
                          ├─ circuit breaker check
                          ├─ request budget check
                          ├─ ScopeValidator.require()   scope check
                          ├─ rate limiter (global)
                          └─ budget += 1
                        ──▶ SafetyEngine.check_destination(ip)
                        ──▶ httpx / dnspython
```

Rules enforced by construction, not by convention:

| Rule | Where it holds |
|---|---|
| Only one scope validator exists | `scope/validator.py` |
| Only one safety engine exists | `engine/safety.py` |
| Every outbound request is authorized | `SafetyEngine.authorize()` |
| Every resolved IP is destination-checked before connect | `HttpProber._resolve_and_check()` |
| Every redirect hop is re-validated | `HttpProber.probe()` |
| No response body is written to disk | `Prober._send()` |
| Sensitive headers are redacted before logging | `logging_config.RedactionFilter` |

Active testing requires **both**:

```yaml
program:
  policy_reviewed: true      # you have read the current program policy
settings:
  active_checks: true        # you explicitly opt in to network testing
```

Either alone is insufficient. There is no CLI flag that bypasses this.

---

## Requirements

- **Python 3.12** or later
- No network access is needed for Phase 1 and Phase 2 commands
- Linux, macOS, or Windows

Runtime dependencies: `httpx`, `dnspython`, `pydantic`, `PyYAML`, `rich`, `idna`.

Development dependencies: `pytest`, `pytest-asyncio`, `respx`, `ruff`, `mypy`.

---

## Installation

### Linux / macOS

```bash
git clone https://github.com/abdullah89255/h1hunter.git
cd h1hunter

python3.12 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

### Windows — PowerShell

```powershell
git clone https://github.com/abdullah89255/h1hunter.git
cd h1hunter

py -3.12 -m venv .venv
.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

### Windows — Command Prompt

```bat
git clone https://github.com/abdullah89255/h1hunter.git
cd h1hunter

py -3.12 -m venv .venv
.venv\Scripts\activate.bat

python -m pip install --upgrade pip
python -m pip install -e ".[dev]"
```

Verify:

```bash
h1hunter --help
h1hunter version
```

---

## Quick start

### 1. Copy the example configs

```bash
cp config/config.example.yaml config/config.yaml
cp config/scope.example.yaml  config/scope.yaml
```

### 2. Edit `config/scope.yaml`

Replace every placeholder with an asset you are **actually authorized to test**.
Set `program.policy_reviewed: true` only after reading the current program
policy on HackerOne.

```yaml
program:
  name: example-program
  policy_reviewed: true
  policy_url: https://hackerone.com/example-program
  authorized_by: you@example.test

allowed:
  - "*.example.com"
  - "https://api.example.net/v1"

excluded:
  - "admin.example.com"

settings:
  passive_recon: true
  active_checks: false       # turn on only when ready for active testing
  max_concurrency: 3
  requests_per_second: 2
  max_requests: 500
  max_response_bytes: 1048576
```

### 3. Validate

```bash
h1hunter scope validate --file config/scope.yaml
h1hunter scope check    --file config/scope.yaml --target https://api.example.net/v1
```

### 4. Plan before you act

```bash
h1hunter plan --file config/scope.yaml \
  --targets example.com attacker.test https://api.example.net/v1/users
```

`plan` prints a table of scope decisions and sends **nothing**.

---

## Configuration

`config/config.yaml` controls *how* the tool runs; `config/scope.yaml`
controls *what* it is allowed to touch.

```yaml
timeout_seconds: 10.0
max_concurrency: 3
requests_per_second: 2.0
max_retries: 2
max_requests: 500
max_response_bytes: 1048576
output_dir: output
user_agent: "h1hunter/0.4 (+authorized-security-research; contact=you@example.test)"

logging:
  level: WARNING        # DEBUG | INFO | WARNING | ERROR | CRITICAL
  format: console       # console | json
  file: null            # any file path here always produces JSON + redacted

enabled_modules: []

passive_sources:
  - name: imported-subdomains
    enabled: true
    options:
      type: text
      path: data/subdomains.txt
  - name: curated-assets
    enabled: false
    options:
      type: file
      path: data/assets.json
```

Secrets are read from the environment only and never written to logs or
reports:

```bash
export H1HUNTER_USER_AGENT="h1hunter/0.4 (+contact=you@example.test)"
export H1HUNTER_SHODAN_API_KEY="..."   # if you later enable remote sources
```

`.env` is git-ignored. Never commit credentials.

---

## Commands

```bash
h1hunter --help
h1hunter version

# Scope
h1hunter scope validate --file config/scope.yaml
h1hunter scope check    --file config/scope.yaml --target https://example.com

# Plan (dry run, no traffic)
h1hunter plan --file config/scope.yaml --targets example.com attacker.test

# Passive recon (local files only)
h1hunter recon passive --file config/scope.yaml --config config/config.yaml \
                       --output output/passive.json --db output/assets.db

# Active recon (require active_checks + policy_reviewed)
h1hunter recon dns   --file config/scope.yaml --targets-file data/hosts.txt --dry-run
h1hunter recon probe --file config/scope.yaml --targets-file data/urls.txt  --dry-run

# Security checks
h1hunter check headers    --file config/scope.yaml --targets-file data/urls.txt --dry-run
h1hunter check tls        --file config/scope.yaml --targets-file data/hosts.txt --dry-run
h1hunter check exposure   --file config/scope.yaml --targets-file data/urls.txt --dry-run
h1hunter check api-docs   --file config/scope.yaml --targets-file data/urls.txt --dry-run
h1hunter check reflection --file config/scope.yaml \
                          --target "https://www.example.com/search?q=x" --dry-run

# Reporting
h1hunter report --input output/headers.json --format markdown --output output/headers.md
h1hunter report --input output/headers.json --format csv      --output output/headers.csv
h1hunter report --input output/headers.json --format json     --output output/headers-2.json
```

### Exit codes

| Code | Meaning |
|------|---------|
| `0`  | Success |
| `1`  | A scope decision denied the request |
| `2`  | Usage, configuration, or storage error |
| `130`| Interrupted by the operator |

### Global flags

| Flag | Purpose |
|------|---------|
| `--config PATH` | Application configuration file |
| `--scope PATH`  | Program scope file |
| `-v`, `-vv`     | Increase log verbosity (`INFO`, `DEBUG`) |
| `--no-color`    | Disable coloured terminal output |
| `--log-format`  | `console` or `json` |
| `--version`     | Print version and exit |

Every active subcommand additionally supports `--dry-run`, `--target`,
`--targets-file`, `--output`, and `--format`.

---

## Architecture

```
cli ──▶ config ──▶ models
 │                  ▲
 ├──▶ scope/parser ──┘
 ├──▶ scope/validator ──▶ scope/matcher ──▶ utils/normalization
 ├──▶ engine/safety ──▶ scope/validator
 │                 ──▶ engine/rate_limiter
 │                 ──▶ utils/networking
 ├──▶ recon/    ──▶ engine/safety
 ├──▶ checks/   ──▶ engine/safety, recon/
 └──▶ reporting/ ──▶ models/
```

See [`docs/architecture.md`](docs/architecture.md) for the full layer diagram,
dependency direction, and extension rules.

---

## Testing

```bash
pytest -q                 # full suite
pytest -q -m safety       # safety regression suite only
ruff check src tests      # lint
mypy                      # static typing
```

**Every test uses mocks.** DNS is stubbed with a plain resolver; HTTP uses
`httpx.MockTransport`; TLS uses stub probes. No test opens a socket.

The `safety` marker selects the regression suite that proves:

- An out-of-scope hostname never reaches the HTTP transport.
- A lookalike host (`example.com.attacker.test`, `notexample.com`) never matches.
- An excluded target is rejected even when a wildcard allows it.
- An out-of-scope redirect is recorded and not followed.
- A missing, empty, or invalid scope file fails closed.
- `active_checks: false` or `policy_reviewed: false` refuses every active command.
- Private, loopback, link-local, and metadata destinations are blocked by default.
- Metadata endpoints stay blocked even with `allow_private_networks: true`.
- The kill switch stops new requests immediately.
- Denied targets do not consume the request budget.
- Sensitive headers never reach a log file or a finding's evidence.
- `--dry-run` produces a plan with zero network traffic.

---

## Project layout

```
h1hunter/
├── pyproject.toml
├── README.md
├── LICENSE
├── .gitignore
├── .env.example
├── config/
│   ├── config.example.yaml
│   └── scope.example.yaml
├── docs/
│   ├── architecture.md
│   ├── scope_safety.md
│   └── usage.md
├── src/h1hunter/
│   ├── cli.py  main.py  config.py  logging_config.py  exceptions.py
│   ├── models/      scope.py  asset.py  finding.py
│   ├── scope/       parser.py  matcher.py  validator.py
│   ├── engine/      rate_limiter.py  safety.py
│   ├── recon/       passive.py  dns.py  http_probe.py  runner.py  registry.py
│   │                providers/{file_source,text_source}.py
│   ├── checks/      base.py  headers.py  tls.py  exposure.py
│   │                api_docs.py  input_reflection.py
│   ├── storage/     database.py  repositories.py
│   ├── reporting/   severity.py  json_report.py  csv_report.py
│   │                markdown_report.py
│   └── utils/       normalization.py  networking.py  deduplication.py
└── tests/
    ├── conftest.py
    ├── fixtures/     sample_scope.yaml  passive_assets.json  passive_hosts.txt
    ├── unit/         6 modules
    └── integration/  4 modules
```

---

## Extending the toolkit

New check modules implement `CheckProtocol`:

```python
# src/h1hunter/checks/my_check.py
from h1hunter.checks.base import CheckContext
from h1hunter.models.finding import Finding, ConfidenceLevel, SeverityLevel

class MyCheck:
    name = "my-check"
    category = "my_category"

    async def run(self, target: str, context: CheckContext) -> list[Finding]:
        async with context.engine.guard(target) as parsed:
            # ... perform the check using context.client / context.resolver
            pass
        return []
```

Rules for new checks:

1. Route every request through `context.engine` and `context.client`.
2. Record the minimum evidence needed to explain the observation.
3. Set `severity_rationale` — never a bare severity value.
4. Default to `VerificationStatus.UNVERIFIED` unless the observation is
   directly observable without contextual judgment.
5. Never modify server state, never download bodies, never access private data.
6. Add a regression test under `tests/unit/` and `tests/integration/`.

Register the module in `cli.py::_instantiate_check` and add a subparser under
the `check` command group.

---

## Known limitations

1. **No remote passive sources ship.** Certificate-transparency, Shodan, and
   SecurityTrails adapters are future work. Passive recon reads local files only.
2. **DNS pre-flight and connection resolution are separate.** A narrow TOCTOU
   window exists between destination check and socket open. The check is
   conservative (all resolved IPs must pass), but IP pinning into the transport
   is not implemented.
3. **`derive_domain` is a two-label heuristic.** It returns `co.uk` as the
   domain for `www.example.co.uk`. Public-suffix integration is future work.
4. **Reflection check replaces only the first query parameter.** Multi-parameter
   endpoints require one invocation per parameter.
5. **TLS weak-protocol detection observes what is negotiated.** A server that
   allows downgrade but prefers TLS 1.3 will not be flagged — this is intentional.
6. **Findings are process-local.** There is no `FindingRepository` yet;
   re-running checks regenerates findings. Cross-module correlation is also absent.
7. **The circuit breaker does not persist across runs.**
8. **No proxy support.** Deliberate: proxies are a common rate-limit and
   access-control evasion vector.

---

## Responsible use

This project is intended for **authorized security testing only**, in
accordance with the published policy of the relevant HackerOne program and all
applicable laws.

Before running any active command:

1. Read the current program policy at its HackerOne page.
2. Confirm the asset you intend to test is explicitly in scope.
3. Note any test-type restrictions (e.g., "no automated scanners", "no rate
   limiting tests").
4. Set `program.policy_reviewed: true` only when those steps are complete.

The toolkit will refuse to run active commands until both `policy_reviewed`
and `active_checks` are `true`, but that refusal is a safety rail — not a
substitute for your own judgment.

**The toolkit never:**

- exploits, fuzzes, or brute forces
- tests credential stuffing or password spraying
- performs denial-of-service or load testing
- accesses private user data
- evades rate limits, proxies, or CAPTCHAs
- downloads suspected secrets or personal information

If you find a vulnerability, follow the program's disclosure process. Do not
publish details before the program has had a reasonable opportunity to remediate.

---

## Contributing

Contributions are welcome. Please:

1. Fork the repository and create a feature branch.
2. Run `pytest -q && pytest -q -m safety && ruff check src tests` before pushing.
3. Add tests for new behaviour — especially anything touching scope, safety,
   destination policy, redirect handling, or secret redaction.
4. Keep new checks non-destructive and evidence-minimal.
5. Update `docs/architecture.md` if you add a new layer or invariant.

Commit messages should describe *why*, not just *what*. Pull requests that
weaken the safety regression suite will not be merged.

---

## License

[MIT](LICENSE) — see the file for the full text.

---

# h1hunter — daily-use commands

Below are the commands you'll actually type. Every active command accepts `--dry-run`, which sends **zero** packets, so run the dry version first when you are unsure.

Set a couple of variables once per shell so the commands stay short:

```bash
cd ~/Desktop/New_Folder/h1hunter
S=config/scope.yaml
C=config/config.yaml
```

---

## 1. Before you do anything

```bash
h1hunter --help
h1hunter version
h1hunter scope validate --file $S
```

`scope validate` prints the program, the number of allowed/excluded entries, and whether active testing is permitted.

---

## 2. Check whether a target is authorized

```bash
h1hunter scope check --file $S --target https://www.tiktok.com
h1hunter scope check --file $S --target api.example.com
h1hunter scope check --file $S --target attacker.test
```

Exit code `0` means allowed, `1` means denied. Useful in a shell script:

```bash
if h1hunter scope check --file $S --target https://www.tiktok.com >/dev/null; then
    echo "in scope"
else
    echo "out of scope"
fi
```

---

## 3. Dry-run planning

```bash
h1hunter plan --file $S --targets www.tiktok.com attacker.test
```

Prints a table of ALLOW/DENY decisions. No traffic.

---

## 4. Passive reconnaissance (local files only)

Create a list of hostnames once:

```bash
mkdir -p data output
printf 'www.tiktok.com\napi.tiktok.com\n' > data/hosts.txt
```

Configure a passive source in `config/config.yaml`:

```yaml
passive_sources:
  - name: seed
    enabled: true
    options:
      type: text
      path: data/hosts.txt
```

Run it:

```bash
h1hunter recon passive --file $S --config $C \
  --output output/passive.json --db output/assets.db
```

Discovered assets are marked `unvalidated`. They are **not** authorized until you run `scope check` on each.

---

## 5. Active reconnaissance

**Always dry-run first:**

```bash
h1hunter recon dns   --file $S --targets-file data/hosts.txt --dry-run
h1hunter recon probe --file $S --target https://www.tiktok.com --dry-run
```

Real runs (require `active_checks: true` and `policy_reviewed: true` in `$S`):

```bash
h1hunter recon dns   --file $S --targets-file data/hosts.txt \
                     --output output/dns.json
h1hunter recon probe --file $S --target https://www.tiktok.com \
                     --output output/probe.json
```

**Bare domains vs URLs:**

| Command | Accepts |
|---|---|
| `recon dns` | `www.tiktok.com` (bare) |
| `recon probe` | `https://www.tiktok.com/` (full URL) |
| `check tls` | `www.tiktok.com` (bare) |
| `check headers` / `exposure` / `api-docs` | `https://www.tiktok.com/` |
| `check reflection` | `https://www.tiktok.com/search?q=x` (must have a query string) |

To convert a bare-host list to URLs:

```bash
awk '{print "https://" $0 "/"}' data/hosts.txt > data/urls.txt
```

---

## 6. Security checks

Dry runs first:

```bash
h1hunter check headers  --file $S --target https://www.tiktok.com --dry-run
h1hunter check tls      --file $S --target www.tiktok.com        --dry-run
h1hunter check exposure --file $S --target https://www.tiktok.com --dry-run
h1hunter check api-docs --file $S --target https://www.tiktok.com --dry-run
h1hunter check reflection --file $S \
  --target "https://www.tiktok.com/search?q=x" --dry-run
```

Real runs, with reports:

```bash
h1hunter check headers --file $S --target https://www.tiktok.com \
  --output output/headers.json

h1hunter check tls --file $S --target www.tiktok.com \
  --output output/tls.md --format markdown

h1hunter check exposure --file $S --target https://www.tiktok.com \
  --output output/exposure.csv --format csv
```

Multiple targets at once:

```bash
h1hunter check headers --file $S --targets-file data/urls.txt \
  --output output/headers.json
```

`--target` is repeatable:

```bash
h1hunter check headers --file $S \
  --target https://www.tiktok.com \
  --target https://api.tiktok.com \
  --output output/headers.json
```

---

## 7. Reports

Turn any findings JSON into another format:

```bash
h1hunter report --input output/headers.json --format markdown --output output/headers.md
h1hunter report --input output/headers.json --format csv      --output output/headers.csv
```

The Markdown report is the one you paste into a HackerOne draft — it includes an authorization notice at the top.

Reports refuse to overwrite existing files. Delete the old one or pick a new name.

---

## 8. Copy-paste cheat sheet

```bash
# Setup once per shell
cd ~/Desktop/New_Folder/h1hunter
S=config/scope.yaml
C=config/config.yaml

# Basics
h1hunter version
h1hunter --help
h1hunter scope validate --file $S
h1hunter scope check    --file $S --target https://www.tiktok.com

# Plan (no traffic)
h1hunter plan --file $S --targets www.tiktok.com

# Passive (local files)
h1hunter recon passive --file $S --config $C --output output/passive.json

# Active (dry run first)
h1hunter recon dns   --file $S --targets-file data/hosts.txt --dry-run
h1hunter recon probe --file $S --target https://www.tiktok.com --dry-run

# Active (real)
h1hunter recon dns   --file $S --targets-file data/hosts.txt --output output/dns.json
h1hunter recon probe --file $S --target https://www.tiktok.com --output output/probe.json

# Checks (dry run first)
h1hunter check headers  --file $S --target https://www.tiktok.com --dry-run
h1hunter check tls      --file $S --target www.tiktok.com        --dry-run
h1hunter check exposure --file $S --target https://www.tiktok.com --dry-run
h1hunter check api-docs --file $S --target https://www.tiktok.com --dry-run
h1hunter check reflection --file $S --target "https://www.tiktok.com/search?q=x" --dry-run

# Reports
h1hunter report --input output/headers.json --format markdown --output output/headers.md
```

---

## 9. Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | A scope decision denied the target |
| `2` | Usage, configuration, or storage error |
| `130` | You pressed Ctrl-C |

Handy for scripts:

```bash
h1hunter scope check --file $S --target https://www.tiktok.com
case $? in
  0) echo "in scope"  ;;
  1) echo "out of scope" ;;
  2) echo "config/usage problem" ;;
esac
```

---

## 10. If something is refused

| Message | Meaning | Fix |
|---|---|---|
| `active checks are disabled` | `active_checks: false` or `policy_reviewed: false` in `$S` | Edit `$S`, set both to `true` |
| `scope file not found` | `$S` does not exist | `cp config/scope.example.yaml config/scope.yaml`, then edit it |
| `target does not match any allowed scope entry` | The target is not in `allowed:` | Add an entry, or pick a different target |
| `target matches exclusion` | The target is in `excluded:` | Remove the exclusion if it's a mistake |
| `refusing to overwrite existing file` | The output path already exists | Delete it or use a different name |
| `http error: Request URL is missing … protocol` | You passed a bare host to an HTTP command | Prefix with `https://` |

---

## Quick rule of thumb

- **Bare hostnames** (`www.tiktok.com`) → `scope check`, `plan`, `recon dns`, `check tls`
- **Full URLs** (`https://www.tiktok.com/`) → `recon probe`, `check headers`, `check exposure`, `check api-docs`
- **Full URLs with a query** (`…/search?q=x`) → `check reflection`

Always `--dry-run` the active commands first. If the plan table looks right, drop `--dry-run` and add `--output`.
