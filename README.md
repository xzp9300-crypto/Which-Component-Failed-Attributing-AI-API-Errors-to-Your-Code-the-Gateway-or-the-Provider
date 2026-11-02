Which Component Failed? Attributing AI API Errors to Your Code, the Gateway, or the Provider
Editorial metadata — remove before publishing
•Slug:/errors/provider-attribution
•Role in cluster: A2 — spoke. Links up to A1 (hub), across to A3.
•Primary keywords:which provider failed api, upstream error vs gateway error, provider returned error attribution, api gateway error troubleshooting, how to tell if the model provider is down
•Intent layers:
￮L1 attribution ("who is at fault") — the highest-intent query in this cluster: the developer is in an incident and needs to route the ticket.
￮L2 isolation ("how do I prove it") — queries about reproducible repro, minimal repro, cross-model comparison.
￮L3 prevention ("how do I build this into my client") — queries about error taxonomy, multi-provider abstraction, observability.
•Primary CTA:/sign-up?lang=en
•Internal links:/errors/429-402-handling, /errors/timeout-streaming, /pricing?lang=en, /ai-api-gateway/
•Word count target: 1,500+ (prose)

Short answer: You cannot attribute a failure from the error message, and you usually cannot attribute it from the status code either — the same 429 means "your key is going too fast" in one layer and "the model's backend is saturated" in another. What does attribute it reliably is a small set of isolation experiments: hold one variable constant, vary one, and see which failures move. Four experiments are enough to separate five failure sources with high confidence, and they take about ten minutes to run.
This page gives you the taxonomy, the experiment matrix, a classifier you can paste into your client, and a probe script that runs all four experiments and prints a verdict. If you only want the code, jump to §5.
One framing note up front, because it saves arguments later: in a gateway chain, "the provider" is usually not a single thing. It is a model id, a routing decision, a provider account, and sometimes a region. Failures attach to different ones of those, and the fix is different for each. The taxonomy below splits accordingly.

1. The five things that can actually fail
#	Source	What it means	Typical statuses	Tell
1	Your request	Malformed, too large, unsupported parameter, bad model id	400, 404, 413, 422	Fails identically every time, on every model, with a body that names the field
2	Your credential / account	Invalid key, revoked key, no balance, key-level cap	401, 402, 403	Fails before any model work; present on every model
3	Gateway platform	Its own rate limiter, routing rules, or an internal fault	429 (with x-ratelimit-*), 500, 503	Affects all models from the same account; carries a gateway rate-limit header
4	Upstream provider	The backend serving your model is saturated, broken, or rate-limiting	429 (no gateway headers), 502, 503, 529	Affects one model or one provider's models; other models fine
5	Model's own limit	Context window, output token cap, reasoning budget consumed	400, or 200 with empty output	Deterministic for that input size; a smaller input succeeds
The differentiator this page is built around is #3 vs #4, because that is the pair with an identical status code and opposite remedies, and because most competing documentation stops at "429 means slow down" — advice that is wrong half the time.
There is a sixth source that is easy to forget and hard to attribute: the network between you and the gateway, and the connection between the gateway and the provider. Those do not produce HTTP statuses at all. They produce truncated bodies, resets, and timeouts, and they are the subject of the streaming and timeout guide. Mentioned here so you do not try to attribute a network failure using an HTTP table.

2. Why the obvious signals fail
The message. As covered in the 429/402 handbook, gateway messages are deliberately generic and, on 5xx, deliberately masked. A message that says "provider returned error" tells you source 4 or 5 and nothing finer. A message that says "invalid token" is reliable. Prose quality is inversely proportional to how much the layer knows.
The status alone.429 spans sources 3 and 4. 400 spans sources 1 and 5. 403 spans source 2 (quota) and policy refusals from source 4. 503 spans 3 and 4. Every ambiguous pair here has a distinguishing signal, but none of them is the number.
The request id. Valuable, but for a different job. A gateway request id (X-Oneapi-Request-Id on Copent, X-Request-Id elsewhere) proves the request reached the gateway and gives support a handle. It does not tell you which layer rejected it. Use it for escalation, not for attribution.
Latency. Weak but real. A rejection in 15ms that never touched a model is a source 1–3 problem. A rejection after 8,000ms of silence is almost always source 4 or 5. Fast-fail versus slow-fail is one of the most informative cheap signals you have, and almost nobody logs it.
So the method is not "read the error better". It is vary one thing at a time and watch what changes.

3. The isolation matrix
Four experiments. Each holds everything constant except a single variable. Run them in this order — the first eliminates the two largest categories immediately.
#	Experiment	Variable changed	What it isolates
E1	Same request on a second, structurally different model (different provider family)	Model / upstream	Source 3 (gateway-wide) vs source 4 (provider-specific)
E2	Same request with a minimal body (one short user message, low max_tokens)	Request payload	Source 1 and source 5 vs everything else
E3	Same request at very low rate (one request, then wait)	Request rate	Source 3 vs source 4 within 429
E4	Same request on a second route/skin (/v1/chat/completions vs /v1/messages)	Wire format	Source 3 (gateway translation) vs source 4
Read the outcomes together, not individually:
•Fails on every model, minimal body, low rate, both routes → source 2 (credential/account) or source 3 (platform). Distinguish by status: 401/402/403 is source 2; 5xx or a 429 carrying gateway headers is source 3.
•Fails only on one model, but succeeds with a minimal body → source 5 (that model's limits), or source 4 with an input-size dependency.
•Fails only on one model, and the minimal body also fails, and low rate also fails → source 4. That model's backend is unhealthy.
•Fails only at high rate, succeeds at one request → source 3 if rate-limit headers are present, source 4 if not.
•Fails on one route only → source 3 (translation layer) or source 1 (you sent a parameter that skin does not support).
•Succeeds everywhere but returned empty content → source 5, specifically a reasoning/max_tokens interaction. Not an error at all.
Four experiments, five sources, and a decision procedure. The value is not in any single row — it is that the rows are mutually exclusive by construction.
Why "second, structurally different model" matters
If you only ever test one model, you cannot separate gateway from provider, ever. The second model must go through a different upstream — a different vendor, not a different size of the same vendor. On a gateway that routes to multiple vendors, an OpenAI-family model next to a Claude-family model next to a Gemini-family model gives you three independent upstreams. Two models from the same vendor share infrastructure and share outages, so comparing them teaches you nothing about source 3.
If both fail at once, the failure is 5xx rather than 4xx, and your minimal request also failed — that is a gateway-platform incident, and no amount of retrying will fix it locally.

4. Signals that are actually attributable
Three fields carry real attribution weight, when the platform emits them.
1. Gateway rate-limit headers. Presence of x-ratelimit-limit-* / x-ratelimit-remaining-* on a 429 means a limiter that knows your quota rejected you, and that limiter sits in front of the model. Absence means you got through to a backend and it refused. This single header check settles the most common ambiguity. (Header naming follows the widely used x-ratelimit-* convention; Azure-hosted OpenAI documents the same six fields plus retry-after-ms.)
2. A typed provider code. Platforms that normalize upstream errors usually preserve the provider's original code in a nested field — provider_code in OpenRouter's error metadata, for example, with the caveat that it is omitted on 500-class errors specifically to avoid leaking upstream internals. If you see a provider code, you have proof the failure came from upstream. If you do not see one on a 500, that is not evidence of a gateway bug — it is documented masking behaviour.
3. Whether the gateway retried before returning. This is the one most people miss, and it is decisive. A gateway that retries upstream failures will, by design, hide some provider failures from you. If it retried three times and still returned the failure, that failure survived three attempts on potentially different backends — which is closer to "this model is down" than "this request was unlucky".
You cannot see the retry count directly. You can infer it from latency: sequential attempts with backoff are additive. On the current New API–family gateways, the retry decision sorts statuses into retryable and skip-retry sets, with 504 and 524 explicitly in the skip set — so a timeout is not retried upstream, while a 429 and a 502 are. That asymmetry is worth knowing when reading latencies. (This is code-level behaviour of the open-source implementation, not a per-account guarantee: an operator can change these ranges. Treat it as a prior, and confirm against your own gateway's observed latencies.)
Counter-intuitive implication: a fast 502 is sometimes more informative than a slow one. Fast means one attempt, likely a hard provider reject. Slow-with-backoff means the gateway worked the problem and still lost.

5. The classifier
This is the piece to paste into your client. It does not guess from messages. It consumes the fields that are attributable, returns an attribution plus a confidence, and deliberately returns unknown rather than inventing a source when the evidence is insufficient — because a wrong attribution sends the ticket to the wrong team, which is worse than an unassigned ticket.
python
# attribution.py - classify an API failure by where it happened.
# Pure stdlib. Designed to be dropped into an existing error handler.

from dataclasses import dataclass, field
from typing import Optional, Dict, Any, List
from enum import Enum

class Source(str, Enum):
    YOUR_REQUEST = "your_request"
    CREDENTIAL_OR_ACCOUNT = "credential_or_account"
    GATEWAY_PLATFORM = "gateway_platform"
    UPSTREAM_PROVIDER = "upstream_provider"
    MODEL_LIMIT = "model_limit"
    NETWORK = "network"
    UNKNOWN = "unknown"

@dataclass
class Failure:
    status: Optional[int]
    headers: Dict[str, str] = field(default_factory=dict)
    body: Dict[str, Any] = field(default_factory=dict)
    latency_ms: Optional[float] = None
    streaming: bool = False
    model: Optional[str] = None
    error_text: str = ""

@dataclass
class Attribution:
    source: Source
    confidence: str          # "high" | "medium" | "low"
    evidence: List[str]
    action: str

GATEWAY_RL_HEADERS = (
    "x-ratelimit-limit-requests",
    "x-ratelimit-remaining-requests",
    "x-ratelimit-limit-tokens",
    "x-ratelimit-remaining-tokens",
)

GATEWAY_ID_HEADERS = ("x-oneapi-request-id", "x-request-id", "x-new-api-version")

# Codes a gateway uses for its own account state rather than an upstream fault.
OWN_QUOTA_CODES = (
    "insufficient_user_quota",
    "pre_consume_token_quota_failed",
    "quota_exhausted",
)

MODEL_LIMIT_PHRASES = (
    "context length",
    "context_length_exceeded",
    "maximum context",
    "max_tokens",
    "too many tokens",
    "string too long",
)

def _h(headers: Dict[str, str], name: str) -> Optional[str]:
    for k, v in headers.items():
        if k.lower() == name:
            return v
    return None

def _has_gateway_rl(headers: Dict[str, str]) -> bool:
    return any(_h(headers, n) is not None for n in GATEWAY_RL_HEADERS)

def _has_gateway_id(headers: Dict[str, str]) -> bool:
    return any(_h(headers, n) is not None for n in GATEWAY_ID_HEADERS)

def _error_fields(body: Dict[str, Any]) -> Dict[str, Any]:
    """Flatten the common envelope shapes into one dict.

    Handles {"error": {...}} (OpenAI/Copent style), nested metadata,
    and flat {"type": ..., "message": ...} bodies.
    """
    if not isinstance(body, dict):
        return {}
    err = body.get("error")
    if isinstance(err, dict):
        out = dict(err)
        meta = err.get("metadata")
        if isinstance(meta, dict):
            out.update({f"meta_{k}": v for k, v in meta.items()})
        return out
    if "message" in body or "type" in body:
        return dict(body)
    return {}

def attribute(f: Failure) -> Attribution:
    fields = _error_fields(f.body)
    text = " ".join(
        str(fields.get(k, "")) for k in ("message", "type", "code", "error_type")
    ).lower() or f.error_text.lower()

    ev: List[str] = []
    status = f.status

    # --- Network layer: no status at all -----------------------------------
    if status is None:
        if f.streaming:
            return Attribution(
                Source.NETWORK, "medium",
                ["no HTTP status; stream terminated"],
                "Treat as partial success: salvage output, do not blindly retry.",
            )
        return Attribution(
            Source.NETWORK, "medium",
            ["no HTTP status at all"],
            "Retry once with backoff; if it repeats, check connectivity, not the model.",
        )

    if _has_gateway_id(f.headers):
        ev.append("gateway request id present")
    else:
        ev.append("no gateway request id in response")

    # --- Credential / account ---------------------------------------------
    if status == 401:
        return Attribution(
            Source.CREDENTIAL_OR_ACCOUNT, "high", ev + ["401 from the auth layer"],
            "Fix the key. Some gateways auto-disable an upstream channel after repeated "
            "401s, so bad-key traffic can have side effects beyond your own account.",
        )

    if status == 402:
        retry_after = _h(f.headers, "retry-after")
        if retry_after:
            return Attribution(
                Source.GATEWAY_PLATFORM, "medium", ev + ["402 with Retry-After"],
                "Transient spend or concurrency hold. Honor Retry-After, then retry.",
            )
        return Attribution(
            Source.CREDENTIAL_OR_ACCOUNT, "high", ev + ["402 without Retry-After"],
            "Credit problem. Top up or raise the key limit; retrying cannot help.",
        )

    if status == 403:
        if any(c in text for c in OWN_QUOTA_CODES):
            return Attribution(
                Source.CREDENTIAL_OR_ACCOUNT, "high", ev + ["403 with an own-quota code"],
                "Your gateway account is out of quota. Top up. Retrying cannot help.",
            )
        return Attribution(
            Source.UPSTREAM_PROVIDER, "medium", ev + ["403 not tied to your own quota"],
            "Policy refusal or permission block. Change the request; do not retry.",
        )

    # --- Request validity --------------------------------------------------
    if status in (400, 413, 422):
        if any(p in text for p in MODEL_LIMIT_PHRASES):
            return Attribution(
                Source.MODEL_LIMIT, "high", ev + ["4xx naming a token or length limit"],
                "Shrink the input or raise the output cap. Deterministic; do not retry.",
            )
        return Attribution(
            Source.YOUR_REQUEST, "high", ev + [f"{status} with a request-shaped error"],
            "Fix the payload or the model id. Deterministic; do not retry.",
        )

    if status == 404:
        return Attribution(
            Source.YOUR_REQUEST, "high", ev + ["404"],
            "Wrong path or unknown model. Check the base path before the key.",
        )

    # --- Rate limiting: the ambiguous one ----------------------------------
    if status == 429:
        if _has_gateway_rl(f.headers):
            return Attribution(
                Source.GATEWAY_PLATFORM, "high",
                ev + ["429 with gateway x-ratelimit-* headers"],
                "Your own rate. Slow down, share one limiter across workers, honor Retry-After.",
            )
        provider_code = fields.get("meta_provider_code") or fields.get("provider_code")
        if provider_code:
            return Attribution(
                Source.UPSTREAM_PROVIDER, "high",
                ev + [f"429 with provider code {provider_code!r}"],
                "Provider capacity, not your rate. Back off with jitter, then fall back.",
            )
        return Attribution(
            Source.UPSTREAM_PROVIDER, "medium",
            ev + ["429 without gateway rate-limit headers"],
            "Most likely provider capacity. Confirm with E1/E3 before changing your own limits.",
        )

    # --- Upstream failure --------------------------------------------------
    if status in (502, 529):
        return Attribution(
            Source.UPSTREAM_PROVIDER, "high",
            ev + [f"{status} = upstream failure or overload"],
            "Retry with jitter and fall back to another model. Do not loop tightly.",
        )

    if status == 503:
        return Attribution(
            Source.UPSTREAM_PROVIDER, "medium", ev + ["503"],
            "No capacity satisfied your routing. Wait briefly, then relax routing or change model.",
        )

    if status in (504, 524):
        return Attribution(
            Source.UPSTREAM_PROVIDER if not f.streaming else Source.NETWORK,
            "medium", ev + [f"{status} timeout"],
            "Not retried upstream by most gateways. Retry once; if it repeats, reduce max_tokens or stream.",
        )

    if status == 500:
        masked = "provider returned error" in text or "internal" in text
        return Attribution(
            Source.UNKNOWN if masked else Source.UPSTREAM_PROVIDER,
            "low", ev + ["500 with a masked message"],
            "Masked by design; provider codes are often stripped on 500. Run E1/E2 before assigning fault.",
        )

    return Attribution(
        Source.UNKNOWN, "low", ev + [f"unclassified status {status}"],
        "Run the isolation matrix before routing this to anyone.",
    )

def explain(f: Failure) -> str:
    a = attribute(f)
    lines = [
        f"attribution : {a.source.value}",
        f"confidence  : {a.confidence}",
        f"action      : {a.action}",
        "evidence    :",
    ]
    lines += [f"  - {e}" for e in a.evidence]
    return "\n".join(lines)
Usage, with a requests-style response:
python
# import requests
# r = requests.post(url, headers=..., json=..., timeout=60)
# print(explain(Failure(
#     status=r.status_code,
#     headers=dict(r.headers),
#     body=r.json() if r.content else {},
#     latency_ms=r.elapsed.total_seconds() * 1000,
#     model="claude-sonnet-5",
# )))
Two design choices worth defending. It returns UNKNOWN on a masked 500 instead of guessing "provider", because that is exactly the case where a guess misroutes an incident. And it rates confidence explicitly, so alerting can page on high-confidence billing failures and merely log low-confidence ones.

6. Probe script: run all four experiments in one sitting
The classifier tells you what a single failure looks like. This script tells you what the pattern looks like — which is what attribution actually requires.
python
#!/usr/bin/env python3
"""
attribute_probe.py - run the four isolation experiments and print a verdict.

Env:
  API_BASE   e.g. https://one.copentai.com
  API_KEY    a valid key
  MODEL_A    primary model, e.g. claude-sonnet-5
  MODEL_B    a model on a DIFFERENT upstream vendor, e.g. gemini-3.6-flash

Optional:
  ANTHROPIC_MODEL  model to hit via /v1/messages (defaults to MODEL_A)
"""
import json
import os
import time
import urllib.error
import urllib.request
from typing import Any, Dict, Tuple

BASE = os.environ["API_BASE"].rstrip("/")
KEY = os.environ["API_KEY"]
MODEL_A = os.environ["MODEL_A"]
MODEL_B = os.environ["MODEL_B"]
ANTHROPIC_MODEL = os.environ.get("ANTHROPIC_MODEL", MODEL_A)
TIMEOUT = 60

PROMPT = "Explain SSE buffering in two sentences."
PROMPT_MIN = "hi"

def call(model: str, *, minimal: bool = False, route: str = "openai") -> Tuple[Any, ...]:
    if route == "openai":
        path = "/v1/chat/completions"
        headers = {"Authorization": f"Bearer {KEY}"}
        payload = {
            "model": model,
            "max_tokens": 16 if minimal else 512,
            "messages": [{"role": "user", "content": PROMPT_MIN if minimal else PROMPT}],
        }
    else:
        path = "/v1/messages"
        headers = {"x-api-key": KEY, "anthropic-version": "2023-06-01"}
        payload = {
            "model": model,
            "max_tokens": 16 if minimal else 512,
            "messages": [{"role": "user", "content": PROMPT_MIN if minimal else PROMPT}],
        }

    headers["Content-Type"] = "application/json"
    req = urllib.request.Request(
        f"{BASE}{path}", data=json.dumps(payload).encode(), headers=headers, method="POST"
    )
    t0 = time.time()
    try:
        with urllib.request.urlopen(req, timeout=TIMEOUT) as r:
            return r.status, dict(r.headers), r.read(800), (time.time() - t0) * 1000
    except urllib.error.HTTPError as e:
        return e.code, dict(e.headers), e.read(800), (time.time() - t0) * 1000
    except Exception as e:
        return None, {}, str(e).encode(), (time.time() - t0) * 1000

def summarize(label: str, res) -> Dict[str, Any]:
    status, headers, body, ms = res
    lower = {k.lower(): v for k, v in headers.items()}
    text = body.decode("utf-8", "replace")
    row = {
        "experiment": label,
        "status": status,
        "ms": round(ms),
        "gateway_rl": any(k.startswith("x-ratelimit-") for k in lower),
        "retry_after": lower.get("retry-after"),
        "gateway_id": lower.get("x-oneapi-request-id") or lower.get("x-request-id"),
        "body": text[:160],
    }
    print(json.dumps(row, indent=2))
    return row

def main():
    print("=== E1: same request, different upstream vendor ===")
    a = summarize(f"E1 primary ({MODEL_A})", call(MODEL_A))
    b = summarize(f"E1 cross-vendor ({MODEL_B})", call(MODEL_B))

    print("\n=== E2: minimal body on the primary model ===")
    minimal = summarize("E2 minimal body", call(MODEL_A, minimal=True))

    print("\n=== E3: single low-rate request on the primary model ===")
    time.sleep(3)
    low_rate = summarize("E3 low rate", call(MODEL_A, minimal=True))

    print("\n=== E4: same model, Anthropic-shaped route ===")
    alt = summarize("E4 alt route", call(ANTHROPIC_MODEL, minimal=True, route="anthropic"))

    rows = [a, b, minimal, low_rate, alt]
    failed = [r for r in rows if r["status"] is None or r["status"] >= 400]

    print("\n=== VERDICT ===")
    if not failed:
        print("No failures observed. Re-run during the incident, or raise load to reproduce.")
        return

    def bad(row):
        return row["status"] is None or row["status"] >= 400

    if all(r["status"] in (401, 402, 403) for r in failed):
        print("Credential/account problem. Every model and every route rejected. "
              "Fix key or balance; not a provider fault.")
    elif bad(a) and bad(b) and bad(minimal):
        if any(r["gateway_rl"] for r in failed):
            print("Gateway platform: rate limiting your account across models. "
                  "Slow down and share a single limiter across workers.")
        else:
            print("Gateway platform or account: all models and all bodies fail alike. "
                  "Check gateway status and account state before blaming a provider.")
    elif bad(a) and not bad(b):
        print(f"Upstream provider specific to {MODEL_A}; cross-vendor {MODEL_B} healthy. "
              "Fall back and open a ticket with the request id.")
    elif not bad(minimal) and bad(a):
        print("Request-shaped: minimal body succeeds, full body fails. "
              "Check input size, context window and max_tokens first.")
    else:
        print("Mixed. Compare the rows above: gateway_rl, retry_after, and latency "
              "(fast fail = rejected early; slow fail = an upstream attempt timed out).")

if __name__ == "__main__":
    main()
What the output lets you say in a ticket: "Cross-vendor model healthy, minimal body healthy on the failing model, failure returns in 40ms with no rate-limit headers." That sentence moves an incident from "our AI is broken" to a named component in one line, and no dashboard gives it to you for free.

7. Reading attribution off latency
Two latency patterns carry more attribution weight than any header:
Fast failure (under ~200ms) with a 4xx. Nothing upstream ran. Whatever rejected you did so on inspection alone: auth, request validation, routing, or a limiter. A 429 that arrives in 30ms did not consult a model.
Slow failure (seconds) with a 5xx or 429. Something upstream was engaged. Either a model backend began work and failed, or the gateway retried several attempts before giving up. Both are source 4. The distinction between "one slow attempt" and "three retried attempts" is visible in the total, because retries with backoff are additive: a 502 at 6s is a different animal from a 502 at 80ms.
Record latency_ms in every error log line. It costs nothing and it is the cheapest attribution signal available. Most teams log status and message and skip the one number that separates layers.

8. What Copent's gateway gives you to attribute with
Measured against the live gateway on 2026-11-02 with an invalid credential, so these describe the rejection path:
•X-Oneapi-Request-Id on every response, also echoed inside the message body. This is the attribution anchor: if it is present, the request reached the gateway. If your client got a bare connection error with no such header, you never got there.
•X-New-Api-Version, confirming the gateway software family — which is what makes the retry and skip-retry behaviour in §4 predictable rather than speculative.
•type as a typed field (new_api_error on rejections we triggered), with code sometimes empty. Do not build logic on code presence.
•Route inventory, confirmable in one probe: /v1/chat/completions, /v1/completions, /v1/responses, /v1/messages, /v1/embeddings, /v1/rerank, /v1/images/generations, /v1/audio/speech, /v1beta/models/{model}:generateContent. A 404 with invalid_request_error means wrong path — which is attribution experiment zero, and it is free.
Not verified yet: whether upstream provider codes are surfaced per model on this gateway, and what your account's retry configuration is. The open-source implementation has a typed provider-code concept and a retry classifier, but a deployment can enable or disable retries per channel, and per-account configuration is not published. Run E1 during a real incident and record what your account actually returns; that observation is worth more than any table, including this one.
Worth repeating from Copent's own pages, because it sets the right expectation: the integration guide tells you not to rely on copied examples that have not been verified against your account, and the Terms describe 99.99% availability as something the platform strives for while providing service "as-is". Both point at the same conclusion — attribution has to be measured on your account, and your client needs its own failure policy regardless of what the gateway promises.

9. Turning attribution into a client contract
Three rules that convert this page into code you keep:
1.One internal error type, normalized at the boundary. Whatever the route — OpenAI-shaped, Anthropic-shaped, Gemini-native — parse into your own Failure with status, headers, flattened body, latency, and streaming flag. Attribution logic then lives in exactly one place.
2.Page on attribution, not on status. A billing attribution pages a human. An upstream-provider attribution increments a counter and triggers fallback. Both may arrive on 429-adjacent codes; they are not the same operational event.
3.Never retry UNKNOWN more than twice. If you could not attribute it, you also cannot argue that retrying helps. Two attempts, then fail with the evidence attached.
And one anti-pattern: do not let attribution logic live in your alerting system only. By the time an alert fires, the response headers are gone. Attribute at the point of failure, log the verdict, alert on the verdict.

Where to go next
For the status-code decision table and the Retry-After parsing the classifier assumes, read the 429/402 handbook. For failures that arrive after a stream has started — where there is no status code to attribute at all — read the streaming and timeout guide, the one case this matrix cannot resolve.
To run these probes against your own account, create one. The scripts need a valid key and a second model on a different vendor, nothing else. Model ids and per-token rates are on the pricing page; per-account limits and retry configuration are only accurate in the console.

Appendix: source notes
•provider_code presence and its omission on 500-class errors, typed error_type vocabulary, and the masking rule: https://openrouter.ai/docs/api_reference/errors-and-debugging
•429 origin split between platform limits and upstream capacity, and X-RateLimit-* appearing only on the error response: https://openrouter.ai/docs/api_reference/limits
•Six x-ratelimit-* fields plus retry-after-ms, three distinct 429 causes, and the 1–10 second burst window: https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/quota
•Gateway retry classification and skip-retry statuses (504, 524), auto-disable on 401, and the typed error vocabulary: New API source, https://github.com/QuantumNous/new-api (setting/operation_setting/status_code_ranges.go, service/relay_error.go, relaykit/types/error.go)
•Copent live headers, route inventory, and error envelopes: direct probes against https://one.copentai.com, 2026-11-02
•Copent availability language and per-account verification guidance: https://one.copentai.com/terms, https://one.copentai.com/ai-model-integration/

Data risk and confidence note
Scene/Argument	Facts	Data Risks and Limitations	Confidence	Verification method
The presence or absence of the x-ratelimit-* header distinguishes between "gateway-specific rate limiting" and "upstream capacity" (429).	Facts verified	Based on two separate documents, OpenRouter and Azure, but not all gateways return this group header.	high	openrouter.ai/docs/api_reference/limits; learn.microsoft.com Azure quota
provider_code was removed in Class 500 errors.	Facts verified	This is an explicit behavior of OpenRouter; other gateways may not follow the same pattern.	high	openrouter.ai/docs/api_reference/errors-and-debugging
The gateway does not retry for 504/524 errors, but retry for 429/502 errors.	Facts to be verified	The source is the default value of the open-source implementation; operators can adjust it according to the channel. It has not been tested on Copent.	middle	github.com/QuantumNous/new-api status_code_ranges.go、service/relay_error.go
The Copent response includes the X-Oneapi-Request-Id, which can be used to determine whether the request reached the gateway.	Facts verified	Tested only with invalid keys, reflecting path rejection.	high	2026-11-02 Direct request to one.copentai.com
Copent's supported route list	Facts verified	The existence of a path was determined based on the difference between 401 and 404 errors, but the completeness of each routing function was not verified.	high	Same as above, path-by-path detection
Does this account expose the upstream provider_code? Is upstream retry enabled?	Facts to be verified	Configuration by account and channel is not disclosed on the public page.	Low	Run E1 in a real incident and record the results.
The attribute_probe.py and attribute.py files are executable.	Facts verified	Not all branches were successfully run on the Copent production account; only syntax and logic self-checks were performed.	middle	Local static checks are required; a single test run is necessary before deployment.
