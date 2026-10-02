---
description: The TailSignal customer portal and data API, with a live demo over a sample of release v1.
---

# Customer portal and API

Customers reach TailSignal's data products two ways: a REST API for their own systems, and a portal built on the same
API for people who want to browse, chart and download. Both check the customer's plan on every call and log usage
for billing.

## Try it

Pick a demo account below: an insurer, a drug manufacturer, a public-health agency, two partner clinics, or the
internal admin view. Each sees only the products in its plan. Every view shows the exact API request, the response,
and the usage it is billed for. Locked products show the refusal the API returns.
[Open the portal full screen](interactive/portal/index.html).

<iframe src="../interactive/portal/" title="TailSignal customer portal demo" style="width:100%;height:1100px;border:1px solid var(--md-default-fg-color--lightest);border-radius:8px" loading="lazy"></iframe>

On this website the portal runs as a demo: requests are answered in the browser from a sample of release v1, with the
same plan checks as the API. Drug-safety signals are limited to pairs with at least 100 reports (2,494 of 53,656);
every other product is complete. When the same page is served by the TailSignal API at `/app`, it calls the live API
with the customer's key.

## Products and plans

| Product | What a customer gets | Insurer | Pharma | Public health | Clinic | Data |
|---|---|---|---|---|---|---|
| Pet Health Index | Prevalence index by species, breed group, market, year and condition, with 90% intervals; illness-cost composite | ✓ | ✓ | | | Simulated partner network |
| Drug-safety signals | Drug–reaction disproportionality (EBGM, EB05) for all dogs, plus breed-specific excesses | | ✓ | ✓ | | Real FDA reports |
| Regional antibiogram | Percent susceptible by region, organism, site and drug | | | ✓ | ✓ | Real FDA NARMS isolates |
| Clinic scorecard | Risk-adjusted complications, deaths, dental charting and stewardship against the network median | | | | Own clinic only | Simulated clinic records |
| Service benchmarks | Revenue per active pet by location and month, with market percentile | | | | | Internal |

## The API

| Endpoint | Returns | Plan needed |
|---|---|---|
| `GET /v1/me` | The key's name, role and entitlements | Any key |
| `GET /v1/releases` | Published releases with row counts | Any key |
| `GET /v1/health-index` | Filter by `species`, `breed_group`, `market`, `year`, `condition`; `composite=true` for the cost index | Health Index |
| `GET /v1/drug-signals` | Filter by `drug`, `event` (contains), `breed_stratum`, `min_eb05` | Drug signals |
| `GET /v1/antibiogram` | Filter by `region`, `organism`, `source` | Antibiogram |
| `GET /v1/clinics/{clinic}/scorecard` | One clinic's scorecard and the network median | Own clinic, or all clinics for internal keys |
| `GET /v1/service-benchmarks` | Filter by `market`, `channel`, `location_id` | Benchmarks |
| `GET /app` | This portal, calling the API with the customer's key | |

Every data endpoint accepts `release=N` to pin a past release, so a customer's analysis is reproducible. Every
successful call is logged with key, endpoint, rows and release version; refused calls return HTTP 403 with the reason.

```bash
curl -H "X-API-Key: demo-pharma" \
  "https://api.tailsignal.example/v1/drug-signals?drug=ivermectin&breed_stratum=mdr1_high&min_eb05=1.5&limit=2"
```

```json
{
  "release": 1,
  "rows": 4,
  "data": [
    {"drug": "ivermectin", "event": "Trembling", "stratum": "mdr1_high", "n": 103,
     "ebgm": 1.95, "eb05": 1.66, "ratio_to_all_dogs_lo": 2.26, "ratio_to_all_dogs": 2.66},
    {"drug": "ivermectin", "event": "Hypersalivation", "stratum": "mdr1_high", "n": 127,
     "ebgm": 1.78, "eb05": 1.54, "ratio_to_all_dogs_lo": 2.05, "ratio_to_all_dogs": 2.38}
  ]
}
```

Herding breeds with the MDR1 variant report trembling with ivermectin 2.7 times as often as dogs overall (lower bound
2.3): the breed-specific signal that pooled analysis hides.

## Run it yourself

```powershell
git clone https://github.com/RonCom/tailsignal && cd tailsignal
uv sync
uv run python -m tailsignal.pipeline            # build the platform
uv run python -m tailsignal.products.release    # publish a release
uv run uvicorn tailsignal.api.app:app           # then open http://127.0.0.1:8000/app
```

Interactive API documentation is at `/docs`. Demo keys are in `config/api_keys.json`; in production they belong in a
secrets store, and usage logs feed billing.

## What a production version adds

- Single sign-on for customer users, with keys issued per integration.
- Rate limits and quotas per plan, and invoices built from the usage log.
- Bulk delivery for data-license customers: Parquet files or a Snowflake data share, versioned like the API.
- An audit trail of which customer received which release, for privacy and contract compliance.
