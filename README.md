# SecondaryDAO Trader API

Documentation for the SecondaryDAO Trader API — programmatic access to real estate token trading, portfolio data, distributions, and market data on Arbitrum One.

**Hosted docs:** [https://secondarydao.github.io/api-docs/](https://secondarydao.github.io/api-docs/)
**Version:** 1.1.0
**Base URL:** `https://api.secondarydao.com` (production) · `http://localhost:5000` (dev)

## Files in this repo

| File | Purpose |
|---|---|
| [`index.html`](index.html) | Landing page for the hosted docs site (GitHub Pages). |
| [`guide.md`](guide.md) | Full human-readable guide — auth, trading model, every endpoint, code examples. |
| [`guide.html`](guide.html) | Rendered version of the guide for the hosted site. |
| [`openapi.yaml`](openapi.yaml) | OpenAPI 3 spec — machine-readable, import into Postman/Insomnia/Swagger UI. |
| `.github/workflows/pages.yml` | Pushes to `main` auto-deploy to GitHub Pages. |

## Auth, in one paragraph

Every authenticated request sends three headers: `X-SD-KEY` (your key ID), `X-SD-TIMESTAMP` (Unix ms, within 30s of server time), and `X-SD-SIGNATURE` (HMAC-SHA256 of `timestamp + METHOD + path + body`, keyed by your 96-char secret). The secret never leaves your machine. Keys are created from the SecondaryDAO web app under Profile → API Keys and the secret is shown once at creation. Full signing examples in Python and Node are in [`guide.md`](guide.md#2-authentication).

## Endpoint map

- **Market data** (public): `/api/properties`, `/api/explorer/properties`, `/api/trading-status/status/{propertyId}`, `/api/order-book/book/{propertyId}`, `/api/merkle/proof/{address}`, `/api/token-stats/*`, `/api/buyout/active-offers/{propertyId}`
- **Portfolio**: `/api/portfolio/{holdings,summary,distributions,performance,trades,income}`
- **Orders & trading**: `/api/trading/orders`, `/api/order-book/{create,my-orders}`, `/api/token-purchase/gasless`
- **Distributions**: `/api/distributions/{unclaimed,history,record-claim}`
- **Buyouts**: `/api/buyout/{check-approval,offer-requirements,my-position,premium-vote-status,voter-status,property-financials}/{propertyId}`
- **Key management**: `/api/api-keys`

Full specs for each endpoint live in [`guide.md`](guide.md) and [`openapi.yaml`](openapi.yaml).

## Quickstart

```bash
# Public — no auth
curl https://api.secondarydao.com/api/properties

# Authenticated — HMAC signed
# See guide.md §2 for a complete signing example in Python and Node
```

## Contributing changes

Edits to this repo go live on the Pages site on push to `main`. Keep `guide.md`, `guide.html`, and `openapi.yaml` in sync — the guide is the source of truth for prose, OpenAPI is the source of truth for request/response shapes.

## Support

Issues and questions: [open an issue](https://github.com/SecondaryDAO/api-docs/issues) or email support via the main site at [secondarydao.com](https://secondarydao.com).
