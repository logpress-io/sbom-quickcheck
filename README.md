# SBOM Impact QuickCheck

One POST, instant CVE impact for your SBOM. Give us a lightweight component list (npm / PyPI today), and get back the exact vulnerabilities and the minimal fixed versions you need to patch. Built for CI pipelines, PR checks, and SRE/AppSec dashboards.

- **API Hub (RapidAPI):** https://rapidapi.com/oslosas/api/sbom-impact-quickcheck
- **Base URL (RapidAPI):** `https://sbom-impact-quickcheck.p.rapidapi.com`
- **Base URL (Direct):** `https://sbom-quickcheck.logpress.io`

> ℹ️ The API is read‑only. No SBOM data is stored, only minimal operational logs (see Terms).

---

## Table of contents
- [Features](#features)
- [Supported ecosystems](#supported-ecosystems)
- [Authentication](#authentication)
  - [Via RapidAPI (recommended)](#via-rapidapi-recommended)
  - [Direct mode (optional)](#direct-mode-optional)
- [SBOM JSON format](#sbom-json-format)
- [Endpoints](#endpoints)
  - [`POST /sbom/impact`](#post-sbomimpact)
  - [`POST /sbom/patchPlan`](#post-sbompatchplan)
  - [`GET /health`](#get-health)
- [Examples](#examples)
  - [cURL](#curl)
  - [JavaScript (fetch)](#javascript-fetch)
  - [GitHub Actions snippet](#github-actions-snippet)
- [Errors](#errors)
- [Rate limits](#rate-limits)
- [Changelog](#changelog)
- [Support](#support)

---

## Features

- ⚡️ **Fast**: ~230–300 ms typical per request (cold start may vary)
- 🎯 **Minimal JSON in/out**: easy to generate from any build system
- 🧩 **Ecosystems**: npm, PyPI (Maven soon)
- 🛠 **Patch plan**: get the *lowest* fixed version to remediate
- 🧱 **Stable contract**: OpenAPI 3 spec & predictable responses

## Supported ecosystems

- `npm` (Node.js)
- `pypi` (Python)
- `maven` (**planned** — same JSON shape)

---

## Authentication

### Via RapidAPI (recommended)

When calling through RapidAPI, set these headers:
```http
X-RapidAPI-Host: sbom-impact-quickcheck.p.rapidapi.com
X-RapidAPI-Key: <your-rapidapi-key>
Content-Type: application/json
```

> You do **not** need an `x-api-key` in RapidAPI mode. The proxy adds the secure bridge header automatically.

### Direct mode (optional)

**Base URL:** `https://sbom-quickcheck.logpress.io`

Add your direct key in the header:
```http
x-api-key: <your-direct-key>
Content-Type: application/json
```
You can request a direct key by contacting support.

---

## SBOM JSON format

A minimal list of components (no lockfile required).

```json
{
  "service": "billing-api",
  "format": "list",
  "components": [
    { "ecosystem": "npm",  "name": "lodash",    "version": "4.17.20" },
    { "ecosystem": "pypi", "name": "requests",  "version": "2.25.0"  }
  ]
}
```

- `ecosystem`: `npm` | `pypi` (| `maven` soon)
- `name`: package name (case-insensitive)
- `version`: exact version string

---

## Endpoints

### `POST /sbom/impact`

Returns vulnerable components with CVEs and severity. Optional query `?minSeverity=LOW|MEDIUM|HIGH|CRITICAL` to filter.

**Request body:** see [SBOM JSON format](#sbom-json-format).

**Response (example):**
```json
{
  "service": "billing-api",
  "impacted": [
    {
      "pkg": "npm:lodash@4.17.20",
      "cves": ["CVE-2020-8203"],
      "severity": "HIGH",
      "fixed": "4.17.21",
      "evidence": "exact-or-range"
    }
  ],
  "summary": {
    "counts": { "CRITICAL": 0, "HIGH": 1, "MEDIUM": 0, "LOW": 0 },
    "total_components": 2,
    "total_impacted": 1
  },
  "version": "vYYYY-MM-DD"
}
```

### `POST /sbom/patchPlan`

Returns the minimal remediation actions (lowest patched versions).

**Response (example):**
```json
{
  "service": "billing-api",
  "actions": [
    {
      "action": "upgrade",
      "ecosystem": "npm",
      "name": "lodash",
      "from": "4.17.20",
      "to": "4.17.21",
      "cves": ["CVE-2020-8203"],
      "rationale": "min-fixed-version"
    }
  ],
  "delta": {
    "upgrades": 1,
    "removals": 0,
    "left_risk_after_min_fix": { "CRITICAL": 0, "HIGH": 0, "MEDIUM": 0, "LOW": 0 }
  },
  "version": "vYYYY-MM-DD"
}
```

### `GET /health`

Returns `{ "ok": true }` when the API is up (no auth required).

---

## Examples

### cURL

**RapidAPI:**
```bash
curl -s --request POST \
  --url https://sbom-impact-quickcheck.p.rapidapi.com/sbom/impact \
  --header 'X-RapidAPI-Host: sbom-impact-quickcheck.p.rapidapi.com' \
  --header 'X-RapidAPI-Key: <YOUR_RAPIDAPI_KEY>' \
  --header 'Content-Type: application/json' \
  --data '{
    "service":"billing-api","format":"list",
    "components":[
      {"ecosystem":"npm","name":"lodash","version":"4.17.20"},
      {"ecosystem":"pypi","name":"requests","version":"2.25.0"}
    ]
  }'
```

**Direct mode:**
```bash
curl -s --request POST \
  --url https://sbom-quickcheck.logpress.io/sbom/impact \
  --header 'x-api-key: <YOUR_DIRECT_KEY>' \
  --header 'Content-Type: application/json' \
  --data '{
    "service":"billing-api","format":"list",
    "components":[
      {"ecosystem":"npm","name":"lodash","version":"4.17.20"},
      {"ecosystem":"pypi","name":"requests","version":"2.25.0"}
    ]
  }'
```

### JavaScript (fetch)

```js
const url = "https://sbom-impact-quickcheck.p.rapidapi.com/sbom/impact";
const payload = {
  service: "billing-api",
  format: "list",
  components: [
    { ecosystem: "npm",  name: "lodash",   version: "4.17.20" },
    { ecosystem: "pypi", name: "requests", version: "2.25.0"  }
  ]
};

const res = await fetch(url, {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "X-RapidAPI-Host": "sbom-impact-quickcheck.p.rapidapi.com",
    "X-RapidAPI-Key": process.env.RAPIDAPI_KEY
  },
  body: JSON.stringify(payload)
});

const data = await res.json();
console.log(data);
```

### GitHub Actions snippet

```yaml
name: SBOM QuickCheck
on:
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build SBOM payload
        run: |
          cat > sbom.json <<'JSON'
          {
            "service":"my-service","format":"list",
            "components":[
              {"ecosystem":"npm","name":"lodash","version":"4.17.20"}
            ]
          }
          JSON

      - name: Call SBOM Impact API (RapidAPI)
        env:
          RAPIDAPI_KEY: ${{ secrets.RAPIDAPI_KEY }}
        run: |
          curl -s --fail --request POST \
            --url https://sbom-impact-quickcheck.p.rapidapi.com/sbom/impact \
            --header 'X-RapidAPI-Host: sbom-impact-quickcheck.p.rapidapi.com' \
            --header "X-RapidAPI-Key: ${RAPIDAPI_KEY}" \
            --header 'Content-Type: application/json' \
            --data @sbom.json | tee result.json

      - name: Fail if HIGH/CRITICAL found
        run: |
          python - <<'PY'
          import json, sys
          j=json.load(open("result.json"))
          counts=j.get("summary",{}).get("counts",{})
          if (counts.get("CRITICAL",0)>0) or (counts.get("HIGH",0)>0):
            print("Found HIGH/CRITICAL issues")
            sys.exit(1)
          PY
```

---

## Errors

| HTTP | Meaning                          | Notes                                  |
|-----:|----------------------------------|----------------------------------------|
| 200  | OK                               | Successful response                    |
| 400  | Bad Request                      | Invalid JSON schema                    |
| 401  | Unauthorized                     | Missing/invalid key                    |
| 413  | Too Many Components              | SBOM exceeds plan limit                |
| 429  | Too Many Requests                | Rate limit exceeded                    |
| 500  | Internal Server Error            | Unexpected error                       |

---

## Rate limits

- Plans define per‑minute / per‑hour quotas (see RapidAPI pricing page).
- Responses include headers such as `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` when applicable.

---

## Changelog

- **2025‑09‑09**: Public beta on RapidAPI (npm & PyPI).

---

## Support

- **Email:** support@logpress.io
- **Issues:** please include endpoint, request ID (if any), and a minimal reproducible SBOM payload.

---

© OSLO SAS. See Terms on the RapidAPI listing.
