# 🔍 Hunter

**Production-grade CLI tool for auditing JavaScript files — finds exposed API keys, secrets, tokens, and internal API endpoints.**

Hunter crawls live web pages *or* scans local source trees, downloads every `<script>` bundle, and runs 35+ regex rules backed by Shannon entropy scoring to surface real credentials while suppressing false positives.

---

## Features

| Capability | Detail |
|---|---|
| **Dual-input** | Target a URL (remote crawl) or a local file / directory (offline audit) |
| **Async engine** | `asyncio` + `httpx` — dozens of scripts fetched concurrently |
| **35+ detection rules** | AWS, GCP, Firebase, Stripe, Slack, SendGrid, GitHub, JWT, DB URIs, private keys, generic secrets, API routes… |
| **Entropy scoring** | Shannon entropy gate filters placeholder strings and framework noise |
| **Multi-format reports** | Rich terminal table · JSON (masked values, metadata) · Standalone dark-themed HTML dashboard |
| **CI-friendly** | Exit code `1` when critical / high findings exist |

---

## Installation

```bash
# Clone & install in editable mode
git clone https://github.com/yourorg/hunter.git
cd hunter
pip install -e .

# Or install dependencies directly
pip install -r requirements.txt
```

Requires **Python 3.10+**.

---

## Usage

### Scan a live URL

```bash
# Crawl a page, discover <script> tags, audit every bundle
hunter scan https://example.com

# Same-origin only (skip CDN / analytics scripts)
hunter scan https://example.com --same-origin

# Export JSON + HTML reports
hunter scan https://example.com --json report.json --html report.html
```

### Scan local files

```bash
# Single file
hunter scan ./dist/bundle.min.js

# Entire directory (recursive)
hunter scan ./src/

# With a stricter entropy threshold
hunter scan ./build/ --entropy 4.0
```

### Options

```
Usage: hunter scan [OPTIONS] TARGET

Arguments:
  TARGET    URL to crawl or local file/directory path to scan  [required]

Options:
  -j, --json PATH            Write JSON report to this path
  -H, --html PATH            Write HTML report to this path
  -e, --entropy FLOAT        Shannon entropy floor               [default: 3.5]
  -c, --concurrency INTEGER  Max concurrent HTTP requests         [default: 20]
  -t, --timeout FLOAT        HTTP request timeout (seconds)       [default: 30]
  -s, --same-origin          Only fetch same-domain scripts
      --no-verify-ssl        Disable TLS certificate verification
  -q, --quiet                Suppress terminal table output
  -V, --version              Show version and exit
```

---

## Project Structure

```
hunter/
├── src/hunter/
│   ├── __init__.py      # Package metadata
│   ├── __main__.py      # python -m hunter entry-point
│   ├── cli.py           # Typer CLI — commands & options
│   ├── config.py        # Tunable runtime settings
│   ├── crawler.py       # Async HTML + script downloader
│   ├── parser.py        # Local file loader + JS beautifier
│   ├── rules.py         # Regex rules, severity, entropy logic
│   ├── scanner.py       # Core scan engine
│   └── reporter.py      # JSON / HTML / terminal reporters
├── requirements.txt
├── pyproject.toml
└── README.md
```

---

## How It Works

1. **Input** — accepts a URL or filesystem path.
2. **Crawl** (URL mode) — fetches the HTML page, extracts `<script src="…">` tags, resolves relative URLs, downloads all scripts concurrently via `httpx.AsyncClient`.
3. **Parse** — applies lightweight beautification to minified bundles so line numbers are meaningful.
4. **Scan** — every rule regex runs against every line; matches pass through:
   - A **length gate** (ignore trivially short strings).
   - An optional **validator** (reject placeholders like `your_api_key`).
   - A **Shannon entropy gate** (demote low-randomness hits).
5. **Report** — findings are sorted by severity, masked, and emitted to one or more outputs.

---

## Example Output

```
┏━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━┓
┃ Sev       ┃ Category               ┃ Rule                         ┃ Value     ┃
┡━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━┩
│ CRITICAL  │ Cloud Provider Keys    │ AWS Access Key ID             │ AKIA****… │
│ HIGH      │ Payment & SaaS Tokens  │ Stripe Live Secret Key        │ sk_l****… │
│ INFO      │ API Endpoints & Routes │ Versioned API Path            │ /api/v1/… │
└───────────┴────────────────────────┴──────────────────────────────┴───────────┘
```

---

## License

MIT
