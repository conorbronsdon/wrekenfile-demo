<div align="center">

# Wrekenfile Demo

Real-world demo of the wrekenfile-converter against the Podcast Index API.

[![GitHub stars](https://img.shields.io/github/stars/conorbronsdon/wrekenfile-demo?style=social)](https://github.com/conorbronsdon/wrekenfile-demo/stargazers)
[![Wreken Spec](https://img.shields.io/badge/Wreken-v2.0.2-purple?style=flat-square)](https://wreken.com)
[![Swytchcode](https://img.shields.io/badge/by-Swytchcode-orange?style=flat-square)](https://www.swytchcode.com/)

[Converter Repo](https://github.com/conorbronsdon/wrekenfile-converter) | [Wreken Spec](https://wreken.com) | [Podcast Index API](https://podcastindex-org.github.io/docs-api/)

</div>

---


## What's here

| File | Description |
|------|-------------|
| `podcastindex-api.json` | Source OpenAPI 3.0.2 spec (from [Podcastindex-org/docs-api](https://github.com/Podcastindex-org/docs-api)) |
| `podcastindex-wrekenfile.yaml` | Converted Wrekenfile v2.0.2 (full API, 3,629 lines) |
| `mini-wrekenfiles/` | 52 standalone mini-wrekenfiles (one per method) |

## How to reproduce

```bash
# Install
npm install wrekenfile-converter

# Convert OpenAPI spec → Wrekenfile
node node_modules/wrekenfile-converter/dist/v2/cli/cli-openapi-to-wrekenfile.js \
  --input demo/podcastindex-api.json \
  --output demo/podcastindex-wrekenfile.yaml

# Generate mini-wrekenfiles for vector DB / LLM context
node node_modules/wrekenfile-converter/dist/v2/cli/cli-mini-wrekenfile-generator.js \
  --input demo/podcastindex-wrekenfile.yaml \
  --output demo/mini-wrekenfiles
```

## Results

**What works:**
- All 50 endpoints converted successfully (52 methods — some endpoints have multiple HTTP methods)
- Clean CANONICAL_ID generation (e.g., `search.byperson.list`, `episodes.byfeedid.list`)
- Auth headers correctly extracted from OpenAPI security schemes (`X-Auth-Key`, `X-Auth-Date`, `Authorization`)
- Parameter LOCATION correctly mapped (query params stay query, no hallucination risk)
- Descriptions preserved with full markdown/examples from the source spec
- Response schemas resolved to typed STRUCTs — 130 `TYPE: STRUCT(...)` references across the full wrekenfile, zero `VOID` returns
- Specific error types per method (`STRUCT(Error400)`, `STRUCT(Error401)`) instead of generic `ANY` — note: these references are currently dangling, see "Known gap" below
- `EXECUTION.MODE: sync` on all REST methods (matches the API's synchronous behavior)
- Mini-wrekenfile generation worked cleanly — each file is standalone and execution-complete
- Fast: full conversion + mini-wrekenfile generation in <1 second

## Issues found (and fixed)

This demo was the external test that surfaced four converter issues. All four were fixed upstream; the output in this repo reflects the fixed behavior.

| Issue | Severity | Fix |
|-------|----------|-----|
| Response types all came back `VOID` — 228 source schemas dropped, `STRUCTS: {}` was empty | High | [conorbronsdon/wrekenfile-converter#1](https://github.com/conorbronsdon/wrekenfile-converter/pull/1) — resolve response-level `$ref`s |
| Generic error descriptions (`Client error (HTTP 400)`) instead of specific error schemas | Medium | [conorbronsdon/wrekenfile-converter#4](https://github.com/conorbronsdon/wrekenfile-converter/pull/4) — use spec-defined error types and descriptions |
| `EXECUTION.MODE: async` on every method, even though Podcast Index is synchronous REST | Low | [conorbronsdon/wrekenfile-converter#4](https://github.com/conorbronsdon/wrekenfile-converter/pull/4) — default REST to `sync` |
| `oneOf`/`anyOf` schemas collapsed to `ANY` | Low | [conorbronsdon/wrekenfile-converter#4](https://github.com/conorbronsdon/wrekenfile-converter/pull/4) — preserve variant structure |

The response schema gap was the big one — it meant an LLM could call the API but couldn't parse results. The fix unlocks the core use case: LLM code generation against a typed API contract.

## Known gap (tracked upstream)

The regeneration surfaced one new class of issue in v2.1.9: the converter emits `STRUCT(...)` references that are never defined under `STRUCTS:` — dangling references for `Error400` (41 refs), `Error401` (41 refs), `categories` (2 refs), `guids`, and `value_batch_byepisodeguid`. A `STRUCT(X)` should always resolve to a definition; here some don't.

This affects typed error handling (since `ERRORS` types are still effectively unresolved) and request-body typing for the batch POST. It does not affect the 200-response typing, which is what PR #1 fixed.

Tracked at [conorbronsdon/wrekenfile-converter#5](https://github.com/conorbronsdon/wrekenfile-converter/issues/5). The demo will be regenerated once that issue is fixed.

## Why this API

The Podcast Index API is a good converter test case because:
- Real production API (not a toy spec)
- Mix of authenticated and unauthenticated endpoints
- Rich response schemas (228 types)
- Multiple parameter types (query strings, path params)
- Well-documented OpenAPI 3.0.2 spec

## About

Demo created by [Conor Bronsdon](https://github.com/conorbronsdon) as an external test of the wrekenfile-converter. All issues filed and fixed upstream.
