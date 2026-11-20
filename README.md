# Endor AURI for Developers — Gemini Enterprise agent

**Find and fix vulnerable open-source dependencies from inside Google Gemini
Enterprise.**

Endor AURI for Developers answers open-source software supply-chain questions and
recommends the exact upgrade that fixes a vulnerable dependency — all inside the
Gemini Enterprise assistant. Answers are read-only and grounded in Endor Labs'
public open-source intelligence, not generated guesses.

## What it does

- **Explain a vulnerability** — severity, CVSS and EPSS scores, a summary, and the
  fixed versions for a CVE or GHSA advisory.
- **Check a package version** — the known vulnerabilities for a specific
  open-source package version.
- **Score a package** — Endor's package score for a version
  (0–10, higher is better).
- **Recommend a safe upgrade** — ranked upgrade options showing which advisories
  each version fixes and whether the jump is a patch, minor, or major change.

Upgrade options render as interactive **"pick an upgrade" cards** in Gemini
Enterprise; selecting one returns a grounded confirmation of what it fixes.

## How to ask it

The agent answers when your message contains a CVE id, a GHSA id, or an Endor
package URL. CVE and GHSA ids are case-insensitive:

| Form | Example |
| ---- | ------- |
| **CVE id** | `What is CVE-2021-44228?` |
| **GHSA id** | `How severe is GHSA-jfh8-c2jp-5v3q?` |
| **Endor package URL** | `Recommend a safe upgrade for mvn://org.apache.logging.log4j:log4j-core@2.14.1` |

Package URLs follow `scheme://name@version` — `npm://`, `pypi://`, `mvn://`,
`go://`, `nuget://`, `gem://`, `cargo://` (for Maven the name is
`group:artifact`, e.g. `mvn://org.apache.logging.log4j:log4j-core@2.14.1`). Write
the scheme in lower-case (`npm://`, not `NPM://`), and keep punctuation off the
end — a trailing `.` becomes part of the version and the lookup misses.

> **Not yet supported:** free-text package names such as *"is log4j 2.14.1
> vulnerable?"*. Ask with an id or a package URL; a plain name returns a short
> help message instead of an answer.

### Example answers

- `What is CVE-2021-44228?` → CRITICAL, CVSS 10.0 — Log4Shell.
- `What is GHSA-jfh8-c2jp-5v3q?` → resolves to the same advisory (CVE-2021-44228).
- `Recommend a safe upgrade for mvn://org.apache.logging.log4j:log4j-core@2.14.1`
  → 7 known vulnerabilities; **recommends 2.25.4** (fixes all 7), with lesser
  options shown fixing fewer.
- `What's the Endor risk score for npm://lodash@4.17.20?` → the package's Endor
  score, with its known vulnerabilities.

## What you need

- An **Endor Labs account** and a **read-only API key**, created at **Settings →
  Access Control → API Keys** on `app.endorlabs.com`. Endor keys are region-bound
  and the agent only calls `api.endorlabs.com`, so a key made on another region
  (e.g. `app.eu.endorlabs.com`) is rejected. The agent reads only Endor's public
  `oss` namespace and exchanges your key for a short-lived, read-only token.
- A **Google Cloud project** to deploy into, and access to Gemini Enterprise.

## Deploy it into your project

Endor AURI runs in **your own Google Cloud project**. For each question it calls
`api.endorlabs.com` to read Endor's public open-source intelligence — no other
Endor-hosted compute is involved.

**Before you install:** enable `config.googleapis.com` in the project, and make
sure your org policy allows the agent's public Cloud Run binding — under Domain
Restricted Sharing the install fails when it verifies the agent card. The
installer bundle README (in the agent repository) has the full
prerequisites and the manual-registration steps.

1. **Install** from the Google Cloud Marketplace listing into your project. The
   installer provisions a short-lived bootstrap VM.
2. The VM deploys the agent as a **Cloud Run** service (HTTPS), then shuts itself
   down. If you supplied a Gemini Enterprise engine at deploy time (the
   `ge_engine_id` variable), it also **registers the agent in Gemini Enterprise**;
   otherwise register it manually against the service's
   `…/.well-known/agent-card.json` (see the installer README).
3. **Add your Endor credential.** The installer already created two Secret Manager
   secrets — `endor-oss-api-key` and `endor-oss-api-secret` — with placeholder
   values, so the agent deploys and registers before you have a key. Add your real
   key and secret as a **new version** of each, then restart the service so the
   running instance picks them up:

   ```bash
   P=<your-project>
   read -rs ENDOR_API_KEY;    echo
   read -rs ENDOR_API_SECRET; echo
   printf '%s' "$ENDOR_API_KEY"    | gcloud secrets versions add endor-oss-api-key    --data-file=- --project="$P"
   printf '%s' "$ENDOR_API_SECRET" | gcloud secrets versions add endor-oss-api-secret --data-file=- --project="$P"
   gcloud run services update <deployment-name>-a2ui --region=<region> --project="$P" \
     --update-env-vars=RESTART_TS=$(date +%s)
   ```

   `read -rs` keeps the key out of your shell history; `printf '%s'` stores it with
   **no trailing newline** (a newline makes Endor reject the key). The values go
   only to Secret Manager — never into the image or deployment state. **Rotating a
   key** is the same three commands.
4. In Gemini Enterprise, **grant your users access** to the agent; it then appears
   in their agent list, ready to use.

## Your credential and data scope

- **Read-only.** The agent explains and recommends; it never modifies code or
  opens pull requests.
- **Public data only.** The agent is pinned in code to Endor's public `oss`
  namespace, so it reads only public open-source intelligence — not your private
  Endor projects or findings. (Your key may itself carry broader tenant access;
  what holds the agent to public data is the fixed namespace, not the token.)
- **Your credential stays yours.** It is sent only to `api.endorlabs.com`,
  exchanged for a short-lived read-only token, and never returned in a response.
- **Honest answers.** When the data has no answer, the agent says so rather than
  inventing a finding or a fix version.

The Cloud Run endpoint is unauthenticated by design — Gemini Enterprise calls it
that way — which is safe because the agent does strictly read-only, public-data
work with validated inputs (no path to tenant data or write calls). The rate cap
is per instance and the endpoint is anonymous, so everyone who can reach the URL
shares that budget — one caller looping it can use up the per-minute allowance
until the window moves.

## Cost

- You pay for the **Google Cloud resources** in your own project: mainly the
  **Cloud Run** service (small — it scales to at most 2 instances), plus the
  stopped installer VM's boot disk (~20 GB) and two Secret Manager secrets that
  remain after install. The agent's Endor queries use the public `oss` namespace.

## Operational limits

Each deployment is capped so it can't overload the Endor API or your Cloud Run
bill:

- Up to ~20 in-flight requests per deployment (2 instances × 10 concurrent).
- At most **60 requests per minute per instance** (up to 120/min per deployment);
  beyond that the agent returns a rate-limit message until the rate drops.
- At most 10 Endor API calls per request, and at most 5 advisory ids per question
  (any extra ids are named as "not checked"). This caps how many ids you name in
  one question — not how many vulnerabilities a package can have (a single package
  may report more; the log4j example above returns 7).
- Requests over 16 KB, or message text over 4 KB, are rejected.

## This version

- Questions are routed by a **deterministic rule engine** that recognizes CVE/GHSA
  ids and Endor package URLs. Plain-language question handling is not part of this
  version.
- Supported ecosystems for package URLs: npm, PyPI, Maven, Go, NuGet, RubyGems,
  and Cargo.

## Documentation & support

- **Documentation:** https://docs.endorlabs.com/
- **Get an API key:** `app.endorlabs.com` → Settings → Access Control → API Keys
- **Support:** your Endor Labs account team

---

## Development

This repository holds the agent service and the Marketplace installer bundle.

- `service/` — the FastAPI A2A/A2UI service (`service.oss.app`) that Gemini
  Enterprise calls.
- `marketplace/vm-installer/` — the Terraform installer bundle.
- `tests/` — the test suite.

Requires Python 3.10+.

```bash
pip install -e ".[dev]"   # install the service and test deps
pytest                    # run the test suite

# Optional pre-commit hooks (gitleaks + the internal-data denylist scan).
# pre-commit isn't in the dev lock yet, so install it first:
pip install pre-commit && pre-commit install
```

Runtime configuration (environment variables):

| Variable          | Values                 | Purpose                            |
| ----------------- | ---------------------- | ---------------------------------- |
| `OSS_ROUTER`      | `rule`                 | question router (rule only)        |
| `OSS_CLIENT`      | `rest`, `mock`         | Endor data source                  |
| `OSS_UI_PROTOCOL` | `a2ui`                 | interactive UI protocol for Gemini |
| `OSS_BRAND_COLOR` | hex string             | brand color for A2UI surfaces      |
