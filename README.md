# emailosint

Fast email-address reconnaissance, written in Rust. One command turns an email
address into a structured intelligence report: mail infrastructure, public
identity, breach membership, and accounts registered across **700+ websites** —
each result carrying an explicit confidence, with coverage reported honestly so
nothing is ever overstated.

```
emailosint scan target@example.com
```

## Why it's different

emailosint is built for investigators who need to trust the output:

- **Deterministic core first.** MX, SPF, DKIM, DMARC, DNSSEC, MTA-STS, BIMI,
  SMTP RCPT + catch-all, disposable/role/free classification, Gravatar, GitHub,
  RDAP and Certificate Transparency — sources that answer reliably, every time.
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

## Install

### Debian / Ubuntu (apt)

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://siddhantbhattarai.github.io/emailosint/pubkey.gpg \
  | sudo tee /etc/apt/keyrings/emailosint.asc > /dev/null

echo "deb [signed-by=/etc/apt/keyrings/emailosint.asc] https://siddhantbhattarai.github.io/emailosint stable main" \
  | sudo tee /etc/apt/sources.list.d/emailosint.list

sudo apt update
sudo apt install emailosint
```

Updates then arrive through `apt upgrade` like any other package. To uninstall:

```bash
sudo apt remove emailosint
sudo rm /etc/apt/sources.list.d/emailosint.list /etc/apt/keyrings/emailosint.asc
```

## Usage

```bash
# Full scan (checks + 700-site enumeration)
emailosint scan target@example.com

# Deterministic checks only, show everything
emailosint scan target@example.com --no-enum --verbose

# Machine-readable output
emailosint scan target@example.com -f json > report.json
emailosint scan target@example.com -f csv  > report.csv

# Username enumeration only
emailosint username johndoe

# Restrict enumeration to categories, tune speed
emailosint scan target@example.com --category social --category coding -j 80

# Inspect the bundled dataset
emailosint data
```

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

```bash
HIBP_API_KEY=xxxx GITHUB_TOKEN=ghp_xxxx emailosint scan target@example.com
```

## Data & updates

The site dataset (`wmn-data.json`, the community-maintained
[WhatsMyName](https://github.com/WebBreacher/WhatsMyName) list) and the
disposable-domains list install to `/usr/share/emailosint/`. Point
`EMAILOSINT_DATA` at another directory to override them without reinstalling:

```bash
EMAILOSINT_DATA=~/.emailosint emailosint scan you@example.com
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

## Credits

- Account-enumeration dataset: [WhatsMyName](https://github.com/WebBreacher/WhatsMyName)
  by Micah Hoffman and contributors, CC BY-SA 4.0.
- Breach data: [Have I Been Pwned](https://haveibeenpwned.com/) by Troy Hunt.

## License

MIT (see `LICENSE`). Bundled dataset is CC BY-SA 4.0.
