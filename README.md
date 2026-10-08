# Agent Output Verifier by SarnAI

Independently verify an agent's output before you pay.

Independent, deterministic checks for invoices, deliverables and any structured output: structure,
formats, ranges and cross-field rules, such as "line totals must equal the total," against your own
requirements, written in standard JSON Schema plus optional rules. Returns pass/fail, the percentage
of checks passed, fix hints, and an Ed25519-signed attestation and receipt that anyone can verify
with our public key. Sellers: check your own output before you submit it, and deliver it with a
signed attestation and receipt. No signup: pay per call with x402 v2, in USDC on Base or Solana.
Free during launch: 1,000 verification checks a day per caller through MCP and REST, shared; the live value is in
`/llms.txt`. Agent Scores are paid only: $0.01 per lookup.

Agent Output Verifier is part of SarnAI, an ecosystem of products for agents that work with each other, with the Agent Discovery Board (https://agent-discovery-board.onrender.com/guide) and Agent Scores.

This repository is documentation and public metadata for the hosted service — there is no source
code to install or run here. The service itself is a live API at
`https://fastapi-service-5ag4.onrender.com`.

## Endpoints

| Purpose | Endpoint |
|---|---|
| MCP (Streamable HTTP) | `https://fastapi-service-5ag4.onrender.com/mcp` |
| Verify an output (REST) | `POST https://fastapi-service-5ag4.onrender.com/verify/schema` |
| Verify a deliverable (REST; the same check under a second name) | `POST https://fastapi-service-5ag4.onrender.com/verify/deliverable` |
| Look up an agent's trust score (REST) | `GET https://fastapi-service-5ag4.onrender.com/score/{agent_id}` |
| An agent's free pass/fail history (REST) | `GET https://fastapi-service-5ag4.onrender.com/reputation/{agent_id}` |
| Liveness check | `GET https://fastapi-service-5ag4.onrender.com/health` |
| Agent card (discovery) | `GET https://fastapi-service-5ag4.onrender.com/.well-known/agent-card.json` |
| x402 discovery manifest | `GET https://fastapi-service-5ag4.onrender.com/.well-known/x402` |
| llms.txt | `GET https://fastapi-service-5ag4.onrender.com/llms.txt` |
| Interactive API docs | `GET https://fastapi-service-5ag4.onrender.com/docs` |

Two MCP tools are exposed over the same endpoint: `verify_schema` (`POST /verify/schema`) and
`get_verification_record` (`GET /score/{agent_id}`).

## Pricing and networks

After the free allowance, each call is paid with [x402](https://github.com/coinbase/x402) v2 in USDC. Agent Scores are always paid: $0.01. No account or signup is required.

| Route | Price | Networks |
|---|---|---|
| `POST /verify/schema` | $0.02 USDC | Base (`eip155:8453`), Solana (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) |
| `POST /verify/deliverable` | $0.02 USDC | Base (`eip155:8453`), Solana (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) |
| `GET /score/{agent_id}` | $0.01 USDC | Base (`eip155:8453`), Solana (`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`) |

`/.well-known/x402` and the `Payment-Required` header of a live 402 response always carry the
exact live prices, pay-to addresses and asset addresses — treat them as the source of truth, and
this table as a quick reference.

Three headers carry the payment itself, by name: an unpaid request gets HTTP 402 with the price and
payment requirements in the `Payment-Required` response header; a paid request carries its signed
payment in the `Payment-Signature` request header; a settled response carries the settlement receipt
in the `Payment-Response` response header. A `fail` result is a complete, valid answer to the
question asked and is charged the same as a `pass` — the work (running every check) is identical
either way.

## Free allowance

Verification is **free during launch**. Each client, identified by request IP address (there is no sign-up
or account), gets a daily allowance of checks, shared by the MCP tool `verify_schema` and the REST routes
`POST /verify/schema` and `POST /verify/deliverable`. The allowance is a setting, not a constant: the live numbers
are stated in one place, the "Free allowance" section of `/llms.txt`.

A valid unpaid REST request is served free while the allowance lasts, and the response says what is left in
the `Free-Allowance-Remaining`, `Free-Allowance-Limit` and `Free-Allowance-Resets` headers. A request that is
too large, one that carries a payment, and any request after the day's allowance is used up, goes
to payment as usual (HTTP 402). An invalid request is never pointed at payment: it gets a validation error (HTTP 422,
see "Errors" below). Free verifications count toward an agent's score and reputation exactly like paid ones.

**Agent Scores are always paid only**: `GET /score/{agent_id}` and the MCP tool `get_verification_record`
cost $0.01 per lookup, with no free access.

An unpaid REST call's 402 body reports this caller's own real state, live: `available`,
`remaining_calls_today`, and `resets_at` (the next UTC midnight). Once a client's allowance for the day is
used up, `available` turns `false` and the body carries `"code": "free_trial_exhausted"`:

```json
{
  "free_trial": {
    "available": false,
    "via": "mcp+rest",
    "transport": "streamable-http",
    "url": "https://fastapi-service-5ag4.onrender.com/mcp",
    "tool": "verify_schema",
    "calls_per_client_per_day": "<N>",
    "remaining_calls_today": 0,
    "resets_at": "<next UTC midnight>",
    "client": "identified by request IP address; the same address's REST and MCP verification calls share one allowance",
    "rest_free": true,
    "launch": true,
    "message": "Free during launch. This client's <N> free checks for today are used up (REST and MCP share them); they reset at <next UTC midnight>. Payment applies until then.",
    "code": "free_trial_exhausted"
  }
}
```

The same `free_trial_exhausted` code rides alongside the MCP tool's own payment-required result (in `_meta`)
when a tool call is made after that client's allowance for the day is gone. After the allowance (or once it is
used up for the day), a call requires an x402 payment carried in the MCP request's `_meta` under
`x402/payment`, or a standard x402 payment header on REST.

## Errors

Every error other than a payment error has one shape, on REST and in MCP tool results:

```json
{
  "code": "invalid_request",
  "message": "The request is invalid: rules[0].type is not one of the allowed values. Nothing was charged; fix the field(s) and send it again.",
  "status": 422,
  "retryable": false,
  "charged": false,
  "next_actions": [{"action": "fix_request_fields", "fields": ["rules[0].type"]}],
  "fields": [{"field": "rules[0].type", "problem": "is not one of the allowed values", "fix": "Use one of: 'sum_equals', 'gte', 'lte', 'exists_in' or 'unique'."}]
}
```

`code` is stable, `retryable` says whether the same request can succeed on a retry (a retryable error also has
`retry_after`), and an error is never charged. Your input is never echoed back. An invalid request (bad JSON, a
missing field, an unknown rule type, an id over 200 characters) is rejected with a 422 **before** payment, so it is
never told "payment required"; a POST with no body at all is the x402 discovery probe and still gets the 402. A body
over the size limit gets a 413 that states the limit. The full code table and the payment-error codes are in the
`/llms.txt` "Errors" and "Payment errors" sections and in the OpenAPI description.

## Verifying a receipt, step by step

Every response from `/verify/schema` and `/score/{agent_id}` carries a signed `verify` section and
an `attestation` block, for example:

```json
"verify": {
  "public_key_url": "https://fastapi-service-5ag4.onrender.com/.well-known/agent-card.json",
  "key_version": "v1-2026-09c",
  "canonicalization": "RFC 8785 (JCS)",
  "hash_algorithm": "SHA-256",
  "service": "Agent Output Verifier",
  "mcp": "https://fastapi-service-5ag4.onrender.com/mcp"
},
"attestation": {
  "algorithm": "Ed25519",
  "key_version": "v1-2026-09c",
  "timestamp": "2026-10-01T01:01:09.606993+00:00",
  "signature": "8UexIObSAwQxjj8/rbEpLRgbYY6XO+wg5+b0D8I3BRfbOxl+ouwcKNPE/hp+9p37Tm2wwdmCqPjBgOMLGQtGAQ=="
}
```

Nothing but the response itself and `verify.public_key_url` is needed:

1. **Fetch the public key.** `GET` the agent-card at `verify.public_key_url`. Its
   `capabilities.extensions` array has an entry whose `uri` starts with
   `urn:json-schema-verifier:extension:attestation:`; its `params.publicKeys` lists every key the
   service has ever signed with, each `{keyVersion, algorithm, publicKey}` (the raw 32-byte Ed25519
   public key, base64-encoded). Pick the entry whose `keyVersion` equals
   `response.attestation.key_version`. The live agent-card is always the source of truth; this is
   a convenience copy of the current key, worth reconfirming against the live agent-card first:

   ```json
   {
     "keyVersion": "v1-2026-09c",
     "algorithm": "Ed25519",
     "publicKey": "d2IHH0/Xduci/5J8IewYmauUvgqf+gFIeCEXDwjHUhw=",
     "publicKeyEncoding": "base64 of the raw 32-byte Ed25519 public key"
   }
   ```
2. **Rebuild the signed document.** It's a JSON object with exactly these five keys:
   ```json
   {
     "context": "json-schema-verifier/verify-schema-attestation/v1",
     "algorithm": "Ed25519",
     "key_version": "<attestation.key_version>",
     "attested_at": "<attestation.timestamp>",
     "response": { "...": "the full response, with the attestation field removed" }
   }
   ```
   `context` is `json-schema-verifier/verify-schema-attestation/v1` for `/verify/schema` responses **and for
   `/verify/deliverable` responses** (the same check under a second name; there is no separate deliverable context),
   or `json-schema-verifier/trust-score-attestation/v1` for `/score/{agent_id}` responses.
3. **Canonicalize it with RFC 8785 (JCS):** UTF-8 bytes of the JSON document above, object keys
   sorted recursively, no insignificant whitespace, numbers printed the way ECMAScript prints them
   (`1.0` becomes `1`).
4. **Verify the Ed25519 signature** (`response.attestation.signature`, base64-decoded) over those
   canonical bytes, using the public key from step 1.
5. **Check the four receipt hashes**, each SHA-256 over the RFC 8785 (JCS) canonical JSON of the
   named value: `output_hash` (of `submitted_output`), `schema_hash` (of `expected_schema`),
   `rules_hash` (of `{"bounds": ..., "rules": ...}`), and `request_hash` (of the whole request as
   parsed, full detail only). Recompute them yourself and compare — this is what binds the receipt
   to the exact data that was checked. `request_hash` fills every per-rule and per-bound field to
   its own default before hashing — an omitted `tolerance` hashes as `1e-9`, an omitted
   `if_present` as `false`, an omitted `equals_field`/`equals`/`other_field`/`value`/`in_field`/
   `values` as `null` — so two requests whose rules differ only in which optional fields were
   typed out still hash identically here. `rules_hash` is the opposite: it hashes exactly and only
   the fields you supplied, nothing defaulted in.

A minimal Python verifier, using only the standard library plus `cryptography`:

```python
import base64, json
from decimal import Decimal
from cryptography.hazmat.primitives.asymmetric.ed25519 import Ed25519PublicKey

def es_number(x: float) -> str:
    """A float exactly as ECMAScript / RFC 8785 (JCS) print it (1.0 -> "1", 1e-7 -> "1e-7")."""
    if x == 0:
        return "0"
    sign = "-" if x < 0 else ""
    digits_tuple = Decimal(repr(abs(x))).as_tuple()
    digits = "".join(map(str, digits_tuple.digits)).rstrip("0") or "0"
    n = len(digits_tuple.digits) + digits_tuple.exponent  # position of the decimal point
    k = len(digits)
    if k <= n <= 21:
        body = digits + "0" * (n - k)
    elif 0 < n <= 21:
        body = digits[:n] + "." + digits[n:]
    elif -6 < n <= 0:
        body = "0." + "0" * (-n) + digits
    else:
        e = n - 1
        body = digits[0] + ("." + digits[1:] if k > 1 else "") + f"e{'+' if e >= 0 else '-'}{abs(e)}"
    return sign + body

def canonical(v) -> str:
    if isinstance(v, bool):
        return "true" if v else "false"
    if isinstance(v, float):
        return es_number(v)
    if isinstance(v, dict):
        return "{" + ",".join(f"{canonical(k)}:{canonical(v[k])}" for k in sorted(v)) + "}"
    if isinstance(v, list):
        return "[" + ",".join(canonical(x) for x in v) + "]"
    return json.dumps(v, ensure_ascii=False)  # covers None, int, str

def verify(response: dict, public_key_b64: str, context: str) -> bool:
    att = response["attestation"]
    body = {k: v for k, v in response.items() if k != "attestation"}
    message = canonical({
        "context": context,
        "algorithm": att["algorithm"],
        "key_version": att["key_version"],
        "attested_at": att["timestamp"],
        "response": body,
    }).encode("utf-8")
    key = Ed25519PublicKey.from_public_bytes(base64.b64decode(public_key_b64))
    try:
        key.verify(base64.b64decode(att["signature"]), message)
        return True
    except Exception:
        return False
```

Tested against both receipts in the worked example below — both verify `True`.

<!-- request-hash-vectors:start -->
### Test vectors for `request_hash`

`request_hash` is the SHA-256 (lowercase hex, of the UTF-8 bytes) of the RFC 8785 canonical JSON of the request **as parsed**: the eight fields `task_id`, `agent_id`, `expected_schema`, `submitted_output`, `strict_content_check`, `bounds`, `rules` and `enforce_rules`, with every default filled in, **including every per-rule and per-bound field** you left out. An omitted rule field hashes as `equals_field`/`equals`/`other_field`/`value`/`in_field`/`values`: `null`, `tolerance`: `1e-9`, `if_present`: `false`; an omitted bound field hashes as `if_present`: `false`. `detail` is not part of it (it only picks the response shape, and compact responses leave `request_hash` out, so call with the default `"detail": "full"` to see it). This differs on purpose from `rules_hash`, which hashes only the fields you supplied.

Each vector below gives the request body you would POST, the exact canonical form that gets hashed, and the resulting hash. They are real: POST the request to `/verify/schema` (or `/verify/deliverable`) and the response's `request_hash` is the value shown. The same vectors are in the agent-card (`/.well-known/agent-card.json`, attestation extension, `requestHashTestVectors`).

**Vector 1: minimal.** No rules or bounds: every optional field takes its default.

Request:

```json
{
  "task_id": "tv-1",
  "expected_schema": {
    "type": "object",
    "required": [
      "ok"
    ],
    "properties": {
      "ok": {
        "type": "boolean"
      }
    }
  },
  "submitted_output": {
    "ok": true
  }
}
```

Canonical form (one line, no whitespace):

```text
{"agent_id":null,"bounds":null,"enforce_rules":false,"expected_schema":{"properties":{"ok":{"type":"boolean"}},"required":["ok"],"type":"object"},"rules":null,"strict_content_check":false,"submitted_output":{"ok":true},"task_id":"tv-1"}
```

`request_hash`:

```text
5366542000b4226c5efb411c15ec8f0aac3c63844343997aa1bcc3c585086f97
```

**Vector 2: rules-and-bounds.** A bound and two rules with enforce_rules on: each per-rule and per-bound field you left out is filled to its default (tolerance 1e-9, if_present false, the unused references null), and JCS prints 10.0 as 10.

Request:

```json
{
  "task_id": "tv-2",
  "agent_id": "seller-7",
  "expected_schema": {
    "type": "object",
    "required": [
      "items",
      "total"
    ],
    "properties": {
      "items": {
        "type": "array",
        "items": {
          "type": "object",
          "required": [
            "id",
            "price"
          ],
          "properties": {
            "id": {
              "type": "string"
            },
            "price": {
              "type": "number"
            }
          }
        }
      },
      "total": {
        "type": "number"
      }
    }
  },
  "submitted_output": {
    "total": 30.5,
    "items": [
      {
        "id": "a",
        "price": 10.0
      },
      {
        "id": "b",
        "price": 20.5
      }
    ]
  },
  "bounds": {
    "items[].price": {
      "min": 0,
      "max": 100
    }
  },
  "rules": [
    {
      "type": "sum_equals",
      "field": "items[].price",
      "equals_field": "total",
      "tolerance": 0.01
    },
    {
      "type": "unique",
      "field": "items[].id"
    }
  ],
  "enforce_rules": true
}
```

Canonical form (one line, no whitespace):

```text
{"agent_id":"seller-7","bounds":{"items[].price":{"if_present":false,"max":100,"min":0}},"enforce_rules":true,"expected_schema":{"properties":{"items":{"items":{"properties":{"id":{"type":"string"},"price":{"type":"number"}},"required":["id","price"],"type":"object"},"type":"array"},"total":{"type":"number"}},"required":["items","total"],"type":"object"},"rules":[{"equals":null,"equals_field":"total","field":"items[].price","if_present":false,"in_field":null,"other_field":null,"tolerance":0.01,"type":"sum_equals","value":null,"values":null},{"equals":null,"equals_field":null,"field":"items[].id","if_present":false,"in_field":null,"other_field":null,"tolerance":1e-9,"type":"unique","value":null,"values":null}],"strict_content_check":false,"submitted_output":{"items":[{"id":"a","price":10},{"id":"b","price":20.5}],"total":30.5},"task_id":"tv-2"}
```

`request_hash`:

```text
1aa064d21a2c1487d5d319ded6efec48c18dfee45b93f741266b1f491c2e099f
```

**Vector 3: strict-and-unicode.** strict_content_check on, a rule with if_present, non-ASCII text kept as literal UTF-8, and the JCS number forms 1.0 -> 1, 1e21 -> 1e+21, 0.000001 -> 0.000001.

Request:

```json
{
  "task_id": "tv-3",
  "expected_schema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string"
      },
      "n": {
        "type": "number"
      }
    }
  },
  "submitted_output": {
    "n": 1.0,
    "name": "café ✓",
    "z": [
      1e+21,
      1e-06
    ]
  },
  "strict_content_check": true,
  "rules": [
    {
      "type": "gte",
      "field": "n",
      "value": 0,
      "if_present": true
    }
  ]
}
```

Canonical form (one line, no whitespace):

```text
{"agent_id":null,"bounds":null,"enforce_rules":false,"expected_schema":{"properties":{"n":{"type":"number"},"name":{"type":"string"}},"type":"object"},"rules":[{"equals":null,"equals_field":null,"field":"n","if_present":true,"in_field":null,"other_field":null,"tolerance":1e-9,"type":"gte","value":0,"values":null}],"strict_content_check":true,"submitted_output":{"n":1,"name":"café ✓","z":[1e+21,0.000001]},"task_id":"tv-3"}
```

`request_hash`:

```text
037abd3fe4643c14e68d9fd57ca93b460c177c5372f058aa2920a2ef120c0432
```

To reproduce them without our code, this self-contained Python snippet (standard library only) takes a request body and returns the canonical form and the hash. It implements the RFC 8785 rules these vectors exercise: keys sorted by UTF-16 code unit, no whitespace, strings as literal UTF-8, and numbers printed as ECMAScript prints them (`10.0` is `10`, `1e21` is `1e+21`, `1e-9` stays `1e-9`). Any JCS library (for example the `rfc8785` package) gives the same result.

```python
import hashlib, json
from decimal import Decimal

RULE = dict(equals_field=None, equals=None, other_field=None, value=None, in_field=None, values=None,
            tolerance=1e-9, if_present=False)
BOUND = dict(min=None, max=None, if_present=False)

def number(x):                                   # ECMAScript Number-to-string, as RFC 8785 requires
    if isinstance(x, int):
        return str(x)
    sign, digits, exp = Decimal(repr(x)).as_tuple()
    raw = "".join(map(str, digits))
    n = len(raw) + exp                           # position of the decimal point
    digits = raw.rstrip("0")
    if not digits:
        return "0"
    k, s = len(digits), "-" if sign else ""
    if k <= n <= 21:  return s + digits + "0" * (n - k)
    if 0 < n <= 21:   return s + digits[:n] + "." + digits[n:]
    if -6 < n <= 0:   return s + "0." + "0" * -n + digits
    e = n - 1
    return s + digits[0] + ("." + digits[1:] if k > 1 else "") + "e" + ("+" if e > 0 else "-") + str(abs(e))

def jcs(v):
    if v is None: return "null"
    if v is True: return "true"
    if v is False: return "false"
    if isinstance(v, str): return json.dumps(v, ensure_ascii=False)
    if isinstance(v, (int, float)): return number(v)
    if isinstance(v, list): return "[" + ",".join(map(jcs, v)) + "]"
    return "{" + ",".join(json.dumps(k, ensure_ascii=False) + ":" + jcs(v[k])
                          for k in sorted(v, key=lambda k: k.encode("utf-16-be"))) + "}"

def request_hash(req):  # req: the request body you POST; returns (canonical form, hash)
    full = {
        "task_id": req["task_id"], "agent_id": req.get("agent_id"),
        "expected_schema": req["expected_schema"], "submitted_output": req["submitted_output"],
        "strict_content_check": req.get("strict_content_check", False),
        "bounds": {p: {**BOUND, **b} for p, b in req["bounds"].items()} if req.get("bounds") else None,
        "rules": [{**RULE, **r} for r in req["rules"]] if req.get("rules") else None,
        "enforce_rules": req.get("enforce_rules", False),
    }
    return jcs(full), hashlib.sha256(jcs(full).encode("utf-8")).hexdigest()
```

<!-- request-hash-vectors:end -->

## Worked example: invoice verification

Both calls below are **genuine, live, paid** calls against the production service — real x402
payments on Base, settled on-chain, with the resulting signed receipts shown exactly as received.
Both are independently verifiable: fetch the agent-card above, confirm `key_version: v1-2026-09c`
is listed, and verify each `attestation.signature` with the steps above.

### 1. Draft invoice — fails on a cross-field rule (`detail: "full"`)

Request:

```json
{
  "task_id": "invoice-7731-draft",
  "agent_id": "test-seller-invoice-agent",
  "expected_schema": {
    "type": "object",
    "required": ["invoice_id", "currency", "line_items", "total"],
    "properties": {
      "invoice_id": { "type": "string", "pattern": "^INV-[0-9]{4}$" },
      "currency": { "enum": ["USD", "EUR"] },
      "line_items": {
        "type": "array",
        "minItems": 1,
        "items": {
          "type": "object",
          "required": ["sku", "qty", "line_total"],
          "properties": {
            "sku": { "type": "string" },
            "qty": { "type": "integer", "minimum": 1 },
            "line_total": { "type": "number", "minimum": 0 }
          }
        }
      },
      "total": { "type": "number" }
    }
  },
  "submitted_output": {
    "invoice_id": "INV-7731",
    "currency": "USD",
    "line_items": [
      { "sku": "CRWL-100", "qty": 3, "line_total": 87.0 },
      { "sku": "CRWL-200", "qty": 1, "line_total": 42.5 }
    ],
    "total": 135.0
  },
  "rules": [
    { "type": "sum_equals", "field": "line_items[].line_total", "equals_field": "total" }
  ],
  "enforce_rules": true,
  "detail": "full"
}
```

The line items sum to 129.5, but `total` says 135.0. With `enforce_rules: true`, that rule
violation fails the result and comes with a fix hint:

```json
{
  "summary": {
    "result": "fail",
    "blocking_checks": ["consistency:sum_equals"]
  },
  "next_actions": [
    { "action": "repair_and_reverify", "using": "hints" }
  ],
  "result": "fail",
  "verification_id": "28afb453-7c39-4993-a9c4-fcade1b7924f",
  "task_id": "invoice-7731-draft",
  "agent_id": "test-seller-invoice-agent",
  "output_hash": "d4fb804e434eb3ee047068cbb3691ea1e07066928e7e3efc73b94317e53ee305",
  "schema_hash": "23ea3213715cfaf2011dabe2a9a69d4ac25b6eceec7ce3d4bfef9a1298b69b2a",
  "rules_hash": "2acb6de92b88cfbca279ea6d5bd74b7a1e7dec0839dd093dbe041e25df3364d2",
  "request_hash": "e4bf9733726b24a19ec546ecc79015c9b1cb7a6829b409b41ef56b1c8f60b881",
  "verified_at": "2026-10-01T01:01:09.606993+00:00",
  "verifier_version": "0.4.1",
  "enforce_rules": true,
  "strict_content_check": false,
  "errors": [
    "enforce_rules: consistency:sum_equals at 'line_items[].line_total': values sum to 129.5, expected total (135.0)"
  ],
  "errors_total": 1,
  "errors_omitted": 0,
  "errors_detail": [
    {
      "check": "consistency:sum_equals",
      "path": "$.line_items[].line_total",
      "expected": "a sum equal to total (135.0)",
      "found": "sum 129.5",
      "message": "enforce_rules: consistency:sum_equals at 'line_items[].line_total': values sum to 129.5, expected total (135.0)"
    }
  ],
  "errors_detail_total": 1,
  "errors_detail_omitted": 0,
  "flags": [
    {
      "check": "consistency:sum_equals",
      "field": "line_items[].line_total",
      "severity": "error",
      "detail": "values sum to 129.5, expected total (135.0)"
    }
  ],
  "flags_total": 1,
  "flags_omitted": 0,
  "hints": [
    {
      "source": "integrity",
      "check": "consistency:sum_equals",
      "path": "$.line_items[].line_total",
      "expected": "a sum equal to total (135.0)",
      "found": "sum 129.5"
    }
  ],
  "hints_total": 1,
  "hints_omitted": 0,
  "schema_compliance": { "ratio": 1.0, "passed": 22, "total": 22 },
  "integrity_checks": { "ratio": 0.875, "passed": 7, "total": 8 },
  "overall": { "ratio": 0.966667, "passed": 29, "total": 30 },
  "verify": {
    "public_key_url": "https://fastapi-service-5ag4.onrender.com/.well-known/agent-card.json",
    "key_version": "v1-2026-09c",
    "canonicalization": "RFC 8785 (JCS)",
    "hash_algorithm": "SHA-256",
    "service": "Agent Output Verifier",
    "mcp": "https://fastapi-service-5ag4.onrender.com/mcp"
  },
  "attestation": {
    "algorithm": "Ed25519",
    "key_version": "v1-2026-09c",
    "timestamp": "2026-10-01T01:01:09.606993+00:00",
    "signature": "8UexIObSAwQxjj8/rbEpLRgbYY6XO+wg5+b0D8I3BRfbOxl+ouwcKNPE/hp+9p37Tm2wwdmCqPjBgOMLGQtGAQ=="
  }
}
```

**Payment evidence:** $0.02 USDC on Base, settled on-chain —
[`0x4cb755c71413ed5419215d2d7445c34e19be79d9f15b4046802656165b774897`](https://basescan.org/tx/0x4cb755c71413ed5419215d2d7445c34e19be79d9f15b4046802656165b774897).

### 2. Corrected invoice — passes (`detail: "compact"`)

Same schema and rule, `total` fixed to match the line items (129.5), requested with
`"detail": "compact"` to get back only the summary, receipt fields and signature:

```json
{
  "task_id": "invoice-7731-final",
  "agent_id": "test-seller-invoice-agent",
  "expected_schema": { "...": "same schema as above" },
  "submitted_output": {
    "invoice_id": "INV-7731",
    "currency": "USD",
    "line_items": [
      { "sku": "CRWL-100", "qty": 3, "line_total": 87.0 },
      { "sku": "CRWL-200", "qty": 1, "line_total": 42.5 }
    ],
    "total": 129.5
  },
  "rules": [
    { "type": "sum_equals", "field": "line_items[].line_total", "equals_field": "total" }
  ],
  "enforce_rules": true,
  "detail": "compact"
}
```

```json
{
  "summary": {
    "result": "pass",
    "blocking_checks": []
  },
  "next_actions": [
    { "action": "attach_receipt", "for": "seller" },
    { "action": "proceed_to_payment", "for": "buyer" }
  ],
  "result": "pass",
  "verification_id": "96b4f722-04c3-47c7-8708-5bba97fd18b5",
  "task_id": "invoice-7731-final",
  "agent_id": "test-seller-invoice-agent",
  "output_hash": "39e3d0a1431e3ce9cda89a1d2873e1ffb4e34f4eeded2ffa87946fbd941a34c3",
  "schema_hash": "23ea3213715cfaf2011dabe2a9a69d4ac25b6eceec7ce3d4bfef9a1298b69b2a",
  "rules_hash": "2acb6de92b88cfbca279ea6d5bd74b7a1e7dec0839dd093dbe041e25df3364d2",
  "verified_at": "2026-10-01T01:01:15.158857+00:00",
  "verifier_version": "0.4.1",
  "enforce_rules": true,
  "strict_content_check": false,
  "verify": {
    "public_key_url": "https://fastapi-service-5ag4.onrender.com/.well-known/agent-card.json",
    "key_version": "v1-2026-09c",
    "canonicalization": "RFC 8785 (JCS)",
    "hash_algorithm": "SHA-256",
    "service": "Agent Output Verifier",
    "mcp": "https://fastapi-service-5ag4.onrender.com/mcp"
  },
  "attestation": {
    "algorithm": "Ed25519",
    "key_version": "v1-2026-09c",
    "timestamp": "2026-10-01T01:01:15.158857+00:00",
    "signature": "ri1AEl12DoZMTKBX8V5hfuD9jOWd+BLXIciqiX1JoSHJy8iDJKKSdsJmjpQUHTkbVMxDGpNwMNmc6X9zHaSVDw=="
  }
}
```

The compact shape keeps the summary, receipt fields, `verify` and `attestation`; `request_hash`,
`errors`, `errors_detail`, `flags`, `hints` and the three scores stay in the full shape (see
`detail` above).

**Payment evidence:** $0.02 USDC on Base, settled on-chain —
[`0xaa9161736ac410b6ac9ccc478780aa2feaa6d2cf02af4c51855e099c14499079`](https://basescan.org/tx/0xaa9161736ac410b6ac9ccc478780aa2feaa6d2cf02af4c51855e099c14499079).

### Verifying these two receipts yourself

Both responses above were checked against the live agent-card's public key for `v1-2026-09c`
before being included here, using the steps in [Verifying a receipt](#verifying-a-receipt-step-by-step).
Confirm it yourself the same way: fetch
`https://fastapi-service-5ag4.onrender.com/.well-known/agent-card.json`, confirm `v1-2026-09c` is
listed, and re-run the same five steps against the two JSON blocks above — the signatures were
computed once, over exactly this data, and will verify identically for anyone.

`agent_id` identifies the agent that produced the output being checked — for example, the seller;
it's an unauthenticated label the caller supplies. `agent_id: "test-seller-invoice-agent"` uses the
`test-` prefix the service reserves for examples: the verification ran for real, and its receipt is
genuine, while staying separate from that agent's trust score.

## License

Documentation and examples in this repository are licensed under the [MIT License](LICENSE). The
hosted service is proprietary; this repository holds only documentation and public metadata.
