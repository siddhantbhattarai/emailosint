# traceosint

Multi-vector identity reconnaissance, written in Rust. Built for investigators: it
correlates an identity across **email, online accounts, phone numbers, and
location artefacts** — each result carrying an explicit confidence, with coverage
reported honestly so nothing is ever overstated.

```
traceosint scan  target@example.com     # email: infra, identity, breaches, 700+ sites + account oracles
traceosint phone +14155552671           # phone: validity, country/region, line type
traceosint ip    203.0.113.9            # IP geolocation: city, ISP/ASN, proxy/hosting flags
traceosint exif  photo.jpg              # EXIF GPS + capture device/time from a shared photo
```

## Commands

```
$ traceosint --help
Multi-vector OSINT: email + phone + account-existence oracles + geo intelligence

Usage: traceosint <COMMAND>

Commands:
  scan      Full scan of an email address (checks + enumeration)
  username  Username-only enumeration across 700+ sites
  phone     Phone-number OSINT: validate + derive country/region/line-type, formats
  ip        IP geolocation: city-level location, ISP/ASN, proxy/hosting flags
  exif      Extract GPS coordinates + capture metadata from a photo's EXIF
  data      Print dataset info (site count, categories, source path)
```

## Capabilities

Everything the tool does, by vector:

**Email — mail infrastructure & identity** (`scan`)
- DNS/mail: MX, SPF, DKIM (common selectors), DMARC, DNSSEC, MTA-STS, BIMI
- SMTP: RCPT deliverability probe + catch-all detection
- Classification: disposable / role / free-provider
- Identity: Gravatar profile + linked accounts, GitHub (public profile email + commit author-email)
- Domain: RDAP registration record, Certificate Transparency (crt.sh)
- Breach membership: Have I Been Pwned *metadata* (~1000 services), when `HIBP_API_KEY` is set

**Email — account-existence oracles** (`scan`, keyed on the email itself)
- Facebook, Instagram, Twitter/X, Pinterest, GitHub, Bitbucket/Atlassian
- Signup-validation technique (`email_is_taken` / `taken:true` / HTTP 422); sends no mail to the target
- Optional proxy routing (`TRACEOSINT_PROXY`) for reliable Meta/Instagram results

**Email/username — site enumeration** (`scan`, `username`)
- 700+ sites via the WhatsMyName dataset, strict two-sided matching, 0% measured false-positive rate

**Phone** (`phone`)
- Validity, country + region, international/national/E.164 formats, line type (mobile / fixed-line / VoIP / toll-free / …) via Google's libphonenumber metadata

**Location intelligence** (`ip`, `exif`)
- IP geolocation: city, region, country, ISP/ASN, timezone, and mobile/proxy/hosting flags
- EXIF: GPS coordinates (with map link) + capture time and camera device from a shared photo

Every result carries a confidence: `[+]` confirmed · `[~]` likely · `[-]` absent · `[?]` inconclusive.

## Why it's different

Most email-OSINT tools inherited holehe's weakness: they scrape ~120 sites,
over half of which now silently fail, and they present ambiguous responses as
hits. traceosint takes the opposite stance:

- **Deterministic core first.** MX, SPF, DKIM, DMARC, DNSSEC, MTA-STS, BIMI,
  SMTP RCPT + catch-all, disposable/role/free classification, Gravatar, GitHub,
  RDAP and Certificate Transparency — sources that answer reliably, every time.
- **Email-keyed account-existence oracles.** Answers "is this *email* registered
  on Facebook / Instagram / Twitter-X / Pinterest / GitHub / Bitbucket?" using each
  platform's own **signup-validation** endpoint (the `email_is_taken` /
  `taken:true` / `HTTP 422` signal) — more reliable than the classic
  forgot-password oracle, and it sends no mail to the target. This is what the old
  username-only profile lookups could never do.
- **Strict, two-sided matching for enumeration.** A site is a hit only when the
  "exists" matcher passes *and* the "missing" matcher fails. Everything else is
  reported as `inconclusive`, not invented as a positive. Measured false-positive
  rate: **0%**.
- **Honest coverage.** Every scan ends with `N confirmed · N not-found ·
  N inconclusive` so an investigator knows exactly what the tool could and could
  not determine.
- **Breach membership, done legally.** Uses Have I Been Pwned's *metadata* API
  (which services an address appears in — the strongest "where is this
  registered" signal, ~1000 services). It never ingests raw stolen dumps, which
  keeps the tool lawful to operate and resell.
- **Rust-fast.** Fully concurrent; a 700-site enumeration completes in a couple
  of minutes, and the deterministic checks in seconds.

### A note on reliability (read this)

The account-existence oracles are as accurate as the IP they run from. Meta
(Facebook/Instagram) and Pinterest rate-limit **per egress IP**, so from a
datacenter/VPS address these often answer `inconclusive` (never a false negative —
absence of a hit is never reported as "not registered"). Twitter/X and GitHub use
cleaner endpoints and usually answer from anywhere. The single biggest reliability
lever is routing through a clean/residential proxy — see `TRACEOSINT_PROXY` below.

## Install

### Debian / Ubuntu (apt)

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://siddhantbhattarai.github.io/traceosint/pubkey.gpg \
  | sudo tee /etc/apt/keyrings/traceosint.asc > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/traceosint.asc] https://siddhantbhattarai.github.io/traceosint stable main" \
  | sudo tee /etc/apt/sources.list.d/traceosint.list

sudo apt update
sudo apt install traceosint
```

Updates then arrive through `apt upgrade` like any other package. To uninstall:

```bash
sudo apt remove traceosint
sudo rm /etc/apt/sources.list.d/traceosint.list /etc/apt/keyrings/traceosint.asc
```

## Usage

```bash
# Full scan (checks + 700-site enumeration)
traceosint scan target@example.com

# Deterministic checks only, show everything
traceosint scan target@example.com --no-enum --verbose

# Machine-readable output
traceosint scan target@example.com -f json > report.json
traceosint scan target@example.com -f csv  > report.csv

# Username enumeration only (Sherlock-style)
traceosint username johndoe

# Restrict enumeration to categories, tune speed
traceosint scan target@example.com --category social --category coding -j 80

# Phone-number intelligence (full +E.164, or national number + --region)
traceosint phone +14155552671
traceosint phone 9812345678 --region NP

# IP geolocation (from a mail header, login record, server log)
traceosint ip 203.0.113.9

# EXIF GPS + capture device/time from a photo the subject shared
traceosint exif ./evidence/photo.jpg

# Inspect the bundled dataset
traceosint data
```

### Routing through a proxy (reliable Meta/Instagram)

`TRACEOSINT_PROXY` (or the standard `HTTPS_PROXY`) routes the account-existence
oracles through a proxy. Point it at a clean residential/mobile proxy and
Facebook/Instagram start answering cleanly instead of rate-limiting:

```bash
TRACEOSINT_PROXY=http://user:pass@residential-proxy:8080 \
  traceosint scan target@example.com --no-enum --verbose
```

### Account-existence oracle coverage

| Platform | Endpoint kind | Notes |
|----------|---------------|-------|
| Twitter / X | dedicated `email_available` | Clean, usually works from any IP. |
| GitHub | `signup_check/email` (422 = taken) | Catches accounts whose profile email is private. |
| Instagram | signup web-create (`email_is_taken`) | Rate-limited per IP; use a proxy for reliability. |
| Facebook | account-recovery identify | Best-effort; Meta challenges hard from datacenter IPs. |
| Pinterest | `EmailExistsResource` | Rate-limited per IP. |
| Bitbucket / Atlassian | signup email validation | Covers Atlassian Cloud (Bitbucket/Jira/Confluence). |

### Output confidence legend

| Symbol | Meaning |
|--------|---------|
| `[+]` confirmed   | Verified true (deterministic, or strict enumeration match). |
| `[~]` likely      | Strong signal, not proof (e.g. accepted SMTP RCPT). |
| `[-]` absent      | Verified false. |
| `[?]` inconclusive| Source blocked/ambiguous — **not** evidence of absence. |

## Optional API keys

| Variable | Enables | Notes |
|----------|---------|-------|
| `HIBP_API_KEY` | Breach membership across ~1000 services | Paid HIBP subscription. Highest-value "where registered" signal. |
| `GITHUB_TOKEN` | Higher-rate GitHub account discovery | Any personal access token; unauthenticated works but is rate-limited. |
| `TRACEOSINT_PROXY` | Route account oracles through a proxy | Not a key — a proxy URL. Residential/mobile proxy = reliable Instagram/Facebook. |

```bash
HIBP_API_KEY=xxxx GITHUB_TOKEN=ghp_xxxx traceosint scan target@example.com
```

## Data & updates

The site dataset (`wmn-data.json`, the community-maintained
[WhatsMyName](https://github.com/WebBreacher/WhatsMyName) list) and the
disposable-domains list install to `/usr/share/traceosint/`. Point
`TRACEOSINT_DATA` at another directory to override them without reinstalling:

```bash
TRACEOSINT_DATA=~/.traceosint traceosint scan you@example.com
```

## Exit codes

`0` something confirmed · `1` error · `2` ran but confirmed nothing — convenient
for scripting and pipelines.

## Legal & ethical

For **authorised investigations only** — your own accounts, consenting clients,
or lawful security/investigative work. Account enumeration may violate some
sites' terms of service. Respect GDPR/CCPA: do not retain target data longer
than necessary and publish a clear acceptable-use policy if you operate this as a
service.

**On location:** traceosint does not — and cannot — track a person's live
location from an email or phone number; no lawful technique does. What it provides
is location *intelligence* about specific artefacts you already lawfully hold: an
IP address (city-level geolocation) or a photo's embedded EXIF GPS. Covert
real-time tracking of a person generally requires legal authority (e.g. a warrant)
that no tool can substitute for.

## Credits

- Account-enumeration dataset: [WhatsMyName](https://github.com/WebBreacher/WhatsMyName)
  by Micah Hoffman and contributors, CC BY-SA 4.0.
- Breach data: [Have I Been Pwned](https://haveibeenpwned.com/) by Troy Hunt.

## License

MIT (see `LICENSE`). Bundled dataset is CC BY-SA 4.0.
