# JUN-054 — curl and HTTP Testing

**VERIQTA | LINUX WORLD — HANDS-ON ENGINEERING LAB**

**Suggested repository path:** `Junior/03-Networking/JUN-054-curl-and-HTTP-Testing/README.md`

| Field | Lab Standard |
|---|---|
| Lab ID | JUN-054 |
| Track | Junior Engineer |
| Topic | HTTP request/response testing, status codes, headers, redirects, timing, and application-layer evidence |
| Difficulty | 3/5 |
| Estimated Duration | 240–270 minutes |
| Operating System | Ubuntu Server 24.04 LTS |
| Required Privileges | Standard user; `sudo` only where explicitly justified |
| Primary Tools | `curl`, `getent`, `ss`, `ip` |
| Lab Style | Guided follow-along engineering investigation |
| Final Deliverable | HTTP Service Investigation Report |
| Next Lab | JUN-055 — SSH Fundamentals |

> **Evidence rule:** Outputs shown in this lab are labeled illustrative, expected, or supplied incident evidence. Your own report must contain what you actually observed. If something is absent, record **Not observed** rather than inventing evidence.

---
# 1. Production Mission

`payments-api.internal.example` is reported unhealthy by an application team. DNS resolves and basic network reachability is not enough: the team needs evidence from the HTTP layer. You will use `curl` to see what the server actually returns, distinguish transport failures from HTTP responses, inspect headers and redirects, control methods and timeouts, measure request timing, and build an evidence trail without modifying the service.

# 2. Learning Workflow

```text
BASELINE → VERIFY curl → RESOLVE TARGET → FIRST GET → STATUS/HEADERS
→ REDIRECTS → METHODS → BODY/HEADERS → TIMEOUTS → TIMING
→ LOCAL CONTROLLED HTTP SERVER → SAFE FAILURES → INCIDENT
→ REFERENCE → DECISION TREE → FINAL CHALLENGE → REPORT
```

# 3. Objectives

You will learn to: use `curl` safely; distinguish DNS, connection, TLS, and HTTP-layer outcomes; interpret status codes; inspect headers; understand redirects; use `-I`, `-i`, `-v`, `-sS`, `-o`, `-w`, `-L`, `--connect-timeout`, and `--max-time`; send explicit methods only when authorized; avoid leaking credentials; test a controlled local HTTP service; and verify the original application symptom.

# 4. Safety and Scope

HTTP requests can have side effects. In this lab, use GET/HEAD against public or approved read-only endpoints and a local controlled server. Do not send POST/PUT/PATCH/DELETE to production endpoints unless explicitly authorized. Never paste tokens, cookies, passwords, private URLs, or sensitive headers into public reports. Do not use `-k/--insecure` as a generic TLS fix.

# 5. Verify Tools and Baseline

```bash
hostname
whoami
command -v curl
curl --version
command -v getent
command -v python3
ip -br addr
ip -4 route show
```

Record the curl version and supported protocols/features shown by your system. Your output may differ.

# 6. Resolve Before Requesting

```bash
getent ahostsv4 example.com
```

Record the returned address. Resolution success proves a name was mapped to an address through the system resolver path; it does not prove TCP connection, TLS negotiation, or HTTP success.

# 7. First HTTP Request

Run:

```bash
curl https://example.com/
```

By default, curl writes the response body to standard output. For troubleshooting, body-only output is often insufficient because you need status and headers.

# 8. Inspect Response Headers

```bash
curl -I https://example.com/
```

`-I` requests headers using a HEAD request where supported. A server can handle HEAD differently from GET, so do not assume identical application behavior.

Now include headers with a normal GET:

```bash
curl -i https://example.com/
```

`-i` includes response headers in output while still performing the normal request.

Record:

```text
STATUS:
SERVER/DATE IF PRESENT:
CONTENT-TYPE:
CONTENT-LENGTH IF PRESENT:
REDIRECT LOCATION IF PRESENT:
```

# 9. Understand HTTP Status Families

- `1xx` informational
- `2xx` successful HTTP processing
- `3xx` redirection
- `4xx` client-side/request/auth/resource class errors
- `5xx` server-side failure class

A `404` proves the HTTP server responded; it is not the same as connection refusal. A `503` is also an HTTP response and therefore different from a TCP connection timeout.

# 10. Separate Body From Diagnostic Output

Discard the body while retaining errors:

```bash
curl -sS -o /dev/null https://example.com/
```

`-s` is silent, `-S` shows errors when silent mode is used, and `-o /dev/null` discards the response body.

Print only the status code:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' https://example.com/
```

This is useful in scripts and runbooks, but status alone is not always enough.

# 11. Observe Redirects

First do not follow redirects:

```bash
curl -i http://example.com/
```

If a redirect is returned, inspect `Location:`. Then, if appropriate:

```bash
curl -L -i http://example.com/
```

`-L` follows redirects. Record both the initial and final behavior. Redirects can cross hosts; be cautious with credentials and authorization headers.

# 12. Verbose Connection Evidence

```bash
curl -v https://example.com/ -o /dev/null
```

Verbose output may show name resolution, connection attempts, TLS negotiation, request headers, and response headers. It can also expose sensitive information. Sanitize before sharing.

Do not confuse `-v` with “more HTTP success.” It provides diagnostic detail.

# 13. Control Timeouts

```bash
curl --connect-timeout 3 --max-time 10 -sS -o /dev/null -w '%{http_code}\n' https://example.com/
```

`--connect-timeout` limits connection establishment. `--max-time` limits the whole operation. Bounded tests are safer for incident work and automation.

# 14. Measure Timing

```bash
curl -sS -o /dev/null \
  -w 'dns=%{time_namelookup} connect=%{time_connect} tls=%{time_appconnect} first_byte=%{time_starttransfer} total=%{time_total}\n' \
  https://example.com/
```

Interpret timings as observations from one request. They do not independently identify root cause. Compare repeated samples and correlate with other evidence.

# 15. Create a Controlled Local HTTP Service

Create a workspace:

```bash
mkdir -p /tmp/veriqta-jun-054/site
cd /tmp/veriqta-jun-054/site
printf 'VERIQTA JUN-054 healthy\n' > index.html
```

Start a local server bound to loopback only:

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Leave it running in this terminal. Open a second terminal/session on the same host.

Verify:

```bash
curl -i http://127.0.0.1:8080/
```

Illustrative response includes `HTTP/1.0 200 OK` or similar and the controlled body.

# 16. Inspect the Listener Without Re-teaching JUN-052

Use your previous skill briefly:

```bash
ss -ltn 'sport = :8080'
```

The purpose is only to correlate the local HTTP endpoint with a listener. Socket inspection was taught in JUN-052; do not turn this section into another socket lab.

# 17. Controlled 404 Failure

Request a missing path:

```bash
curl -i http://127.0.0.1:8080/does-not-exist
```

Observe the HTTP status. The TCP connection and HTTP exchange can succeed even though the requested resource does not exist.

Record:

```text
TRANSPORT REACHED SERVER?:
HTTP STATUS:
RESOURCE SUCCESSFUL?:
```

# 18. Controlled Connection-Refused Failure

Stop the local Python server with `Ctrl+C` in the server terminal. Verify the listener is gone:

```bash
ss -ltn 'sport = :8080'
```

Then:

```bash
curl --connect-timeout 2 http://127.0.0.1:8080/
```

Record the exact curl error. Compare this with the earlier 404. One is an HTTP response; the other fails before HTTP response processing.

# 19. Restart the Controlled Server

Restart it exactly as before and verify `curl` returns the expected content. This is a controlled recovery, not a recommendation to restart unknown production services.

# 20. Failure Lab — DNS vs HTTP

```bash
curl --connect-timeout 2 https://does-not-exist.invalid/
```

Expected: resolution failure. Compare with a request to a resolvable host that returns an HTTP status. State which layer failed.

# 21. Failure Lab — Timeout vs Refusal

Supplied evidence A:

```text
curl: (7) Failed to connect ... Connection refused
```

Supplied evidence B:

```text
curl: (28) Connection timed out ...
```

Refusal commonly means a connection reached a host/path that actively rejected it or no listener accepted it. Timeout means completion was not observed within the limit. Neither message alone proves the complete root cause.

# 22. Failure Lab — HTTP 503

Supplied evidence:

```text
HTTP/1.1 503 Service Unavailable
Retry-After: 30
```

What is known? An HTTP-speaking endpoint responded with a server-side unavailable status. Do not report this as “network unreachable.” Investigate application/upstream/load-balancer health.

# 23. Failure Lab — Redirect Surprise

Supplied evidence:

```text
HTTP/1.1 301 Moved Permanently
Location: https://api.example.internal/v2/
```

A health checker that does not follow redirects may mark this differently from a browser. Determine expected behavior before adding `-L` blindly.

# 24. Production Incident — Health Endpoint Failure

**Impact:** deployment validation reports `payments-api` unhealthy.

**Expected:** `https://payments.internal.example/health` returns `200` quickly.

### Evidence 1

```text
getent ahostsv4 payments.internal.example
10.40.12.18 ...
```

### Evidence 2

```text
curl --connect-timeout 2 --max-time 5 -i https://payments.internal.example/health
HTTP/1.1 503 Service Unavailable
content-type: application/json

{"status":"unavailable","dependency":"ledger"}
```

What does this prove? DNS, connection/TLS/HTTP progressed far enough to receive an application/gateway response. It does not prove the ledger dependency is actually the root cause; the body is evidence supplied by the service.

### Hypotheses

Create at least three: ledger dependency unavailable; application configuration points to wrong ledger endpoint; health endpoint itself is misconfigured; load balancer routes to an unhealthy backend.

### Evidence 3

```text
curl -sS -o /dev/null -w 'code=%{http_code} connect=%{time_connect} total=%{time_total}\n' https://payments.internal.example/health
code=503 connect=0.004812 total=0.019441
```

Fast 503 responses make a generic network timeout less likely.

### Evidence 4

Application owner supplies:

```text
ledger endpoint configured as ledger-old.internal.example:9443
approved endpoint is ledger.internal.example:9443
```

Now the configuration mismatch is supported as root cause.

### Authorized remediation

The application owner corrects the persistent configuration through the approved deployment system. Do not edit live production configuration ad hoc.

### Verification

Repeat the exact `/health` request and the original deployment validation. Confirm `200`, expected body, acceptable timing, and monitoring recovery.

### Prevention

Configuration validation, deployment-time health checks, dependency endpoint tests, and configuration-as-code review.



# Extended Follow-Along Engineering Practice — HTTP Request/Response Evidence

This block deepens the same lab scope. Work through it in order; do not treat it as optional reading. For every command, record your own result and write one sentence stating what the evidence proves and one sentence stating what it does not prove.

## Practice 1 — Capture status without body

When the body is large, isolate status while retaining errors.

Run the following only on the authorized lab host/target:

```bash
curl -sS -o /dev/null -w 'status=%{http_code}\n' https://example.com/
```

A numeric status proves an HTTP response was received; it does not prove the response body is semantically correct.

Record your evidence:

```text
STATUS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 2 — Capture headers separately

Headers often explain caching, content type, redirects, or server behavior.

Run the following only on the authorized lab host/target:

```bash
curl -sS -D /tmp/jun054.headers -o /dev/null https://example.com/
sed -n '1,20p' /tmp/jun054.headers
```

`-D` writes response headers. Treat header files as potentially sensitive on internal services.

Record your evidence:

```text
STATUS LINE:
CONTENT-TYPE:
LOCATION IF ANY:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 3 — Compare HEAD and GET

HEAD can differ from GET, so compare deliberately.

Run the following only on the authorized lab host/target:

```bash
curl -sS -I https://example.com/
curl -sS -o /dev/null -w 'GET=%{http_code}\n' https://example.com/
```

If behavior differs, do not assume one method represents the other.

Record your evidence:

```text
HEAD STATUS:
GET STATUS:
DIFFERENCE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 4 — Observe redirect chain

A redirect can be expected architecture or an incident symptom.

Run the following only on the authorized lab host/target:

```bash
curl -sS -o /dev/null -D - http://example.com/
curl -sS -L -o /dev/null -w 'final=%{url_effective} code=%{http_code}\n' http://example.com/
```

Record initial response and final effective URL separately.

Record your evidence:

```text
INITIAL STATUS/LOCATION:
FINAL URL:
FINAL STATUS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 5 — Measure DNS and connect time

Timing fields help localize delay phases without proving root cause.

Run the following only on the authorized lab host/target:

```bash
curl -sS -o /dev/null -w 'dns=%{time_namelookup} connect=%{time_connect} total=%{time_total}\n' https://example.com/
```

A high DNS time suggests where to investigate next, but one sample is not a diagnosis.

Record your evidence:

```text
DNS TIME:
CONNECT TIME:
TOTAL:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 6 — Measure TLS and first byte

HTTPS adds TLS negotiation and server response time.

Run the following only on the authorized lab host/target:

```bash
curl -sS -o /dev/null -w 'tls=%{time_appconnect} first=%{time_starttransfer} total=%{time_total}\n' https://example.com/
```

Compare phases across several samples before attributing latency.

Record your evidence:

```text
TLS TIME:
FIRST BYTE:
TOTAL:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 7 — Bound a health check

Runbooks should not hang indefinitely.

Run the following only on the authorized lab host/target:

```bash
curl --connect-timeout 2 --max-time 5 -sS -o /dev/null -w '%{http_code}\n' https://example.com/
```

The bounds are part of the test definition and should be documented.

Record your evidence:

```text
CONNECT TIMEOUT:
MAX TIME:
RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 8 — Inspect verbose stages

Use verbose mode when you need stage-level evidence.

Run the following only on the authorized lab host/target:

```bash
curl -v --connect-timeout 3 https://example.com/ -o /dev/null
```

Identify resolution, connection, TLS, request, and response lines. Sanitize before sharing.

Record your evidence:

```text
LAST SUCCESSFUL STAGE:
FIRST FAILURE/ANOMALY:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 9 — Test a controlled local path

A local server gives you known expected content.

Run the following only on the authorized lab host/target:

```bash
cd /tmp/veriqta-jun-054/site
curl -sS -i http://127.0.0.1:8080/
```

Compare expected controlled content with status and headers.

Record your evidence:

```text
STATUS:
BODY EXPECTED?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 10 — Differentiate missing resource

A 404 is application-layer evidence, not a transport failure.

Run the following only on the authorized lab host/target:

```bash
curl -sS -i http://127.0.0.1:8080/not-here
```

Record that HTTP exchange succeeded even though resource lookup failed.

Record your evidence:

```text
STATUS:
TRANSPORT REACHED HTTP?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 11 — Record response size

Response size can reveal unexpected error pages or truncation.

Run the following only on the authorized lab host/target:

```bash
curl -sS -o /dev/null -w 'code=%{http_code} bytes=%{size_download}\n' https://example.com/
```

Size is supporting evidence; content semantics still matter.

Record your evidence:

```text
STATUS:
BYTES:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 12 — Build an HTTP evidence ladder

Combine resolution and HTTP evidence without changing service state.

Run the following only on the authorized lab host/target:

```bash
getent ahostsv4 example.com
curl --connect-timeout 3 --max-time 10 -sS -o /dev/null -w 'code=%{http_code} connect=%{time_connect} total=%{time_total}\n' https://example.com/
```

Write separate conclusions for resolution, connection timing, and HTTP status.

Record your evidence:

```text
RESOLUTION:
HTTP STATUS:
TIMING:
NEXT TEST:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.




# Engineering Scenario Drill Bank

Use these as short incident rehearsals. For each scenario, do **not** jump directly to a fix. Write: symptom, expected state, first evidence to collect, two plausible hypotheses, one discriminating test for each hypothesis, safe remediation owner, verification of the original operation, and one prevention control.

## Drill 1 — Dns Resolution Failure

**Scenario:** The production symptom is **DNS resolution failure**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: DNS resolution failure
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 2 — Connection Refused

**Scenario:** The production symptom is **connection refused**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: connection refused
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 3 — Connection Timeout

**Scenario:** The production symptom is **connection timeout**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: connection timeout
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 4 — Tls Certificate Verification Failure

**Scenario:** The production symptom is **TLS certificate verification failure**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: TLS certificate verification failure
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 5 — Http 301 Unexpected Redirect

**Scenario:** The production symptom is **HTTP 301 unexpected redirect**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: HTTP 301 unexpected redirect
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 6 — Http 401 Authentication Required

**Scenario:** The production symptom is **HTTP 401 authentication required**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: HTTP 401 authentication required
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 7 — Http 404 Wrong Path

**Scenario:** The production symptom is **HTTP 404 wrong path**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: HTTP 404 wrong path
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 8 — Http 503 Dependency Unavailable

**Scenario:** The production symptom is **HTTP 503 dependency unavailable**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: HTTP 503 dependency unavailable
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 9 — Http 200 With Unexpected Content

**Scenario:** The production symptom is **HTTP 200 with unexpected content**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: HTTP 200 with unexpected content
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.

## Drill 10 — Slow Time-To-First-Byte

**Scenario:** The production symptom is **slow time-to-first-byte**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: slow time-to-first-byte
IMPACT:
EXPECTED STATE:
KNOWN EVIDENCE:
UNKNOWN:
HYPOTHESIS 1:
TEST 1:
HYPOTHESIS 2:
TEST 2:
ROOT CAUSE: not established until evidence supports it
AUTHORIZED REMEDIATION OWNER:
RECOVERY VERIFICATION:
PREVENTION:
```

Your first test must stay inside this lab's scope. If the next useful test belongs to a later lab or another team, state that boundary explicitly rather than pretending the current tool can answer every question.


# 25. curl Command-and-Flag Reference

| Command | Purpose |
|---|---|
| `curl URL` | Perform a default request and print body |
| `curl -I URL` | Request headers with HEAD where supported |
| `curl -i URL` | Include response headers with normal output |
| `curl -v URL` | Verbose connection/protocol diagnostics |
| `curl -sS URL` | Quiet progress but still show errors |
| `curl -o FILE URL` | Write body to a file/device |
| `curl -w FORMAT URL` | Print selected transfer metadata |
| `curl -L URL` | Follow redirects |
| `curl --connect-timeout N URL` | Bound connection establishment |
| `curl --max-time N URL` | Bound total operation time |

| Option | Meaning / caution |
|---|---|
| `-I` | HEAD request; may differ from GET behavior |
| `-i` | Include response headers |
| `-v` | Verbose; sanitize sensitive output |
| `-sS` | Silent progress plus errors |
| `-o /dev/null` | Discard body |
| `-w` | Write metrics/status after transfer |
| `-L` | Follow redirects; consider credential boundaries |
| `--connect-timeout` | Connection phase bound |
| `--max-time` | Whole-operation bound |

# 26. HTTP Troubleshooting Decision Tree

```text
HTTP CHECK FAILS
      |
      v
NAME RESOLVES?
 |          |
NO         YES
 |          v
DNS      CONNECTION ESTABLISHED?
          |             |
         NO            YES
          |             v
   route/port/TLS    HTTP RESPONSE?
                        |      |
                       NO     YES
                        |      v
                   protocol   STATUS/BODY/HEADERS
                   failure       |
                                 v
                         compare expected behavior
```

| Observation | Next reasoning |
|---|---|
| Could not resolve host | DNS/NSS path |
| Connection refused | listener/port/path policy evidence needed |
| Connection timeout | path/filter/service evidence needed |
| TLS verification failure | certificate/name/trust investigation |
| 404 | HTTP reached endpoint; resource/path issue |
| 503 | HTTP endpoint responded unavailable; investigate service/upstream |
| 301/302 | Determine whether redirect is expected |

# 27. Independent Final Challenge — HTTP Service Investigation

Investigate an authorized endpoint with this symptom: “API health check fails after deployment.” You must establish name resolution, bounded request behavior, HTTP status, important headers, redirect behavior, and timing. Use verbose mode only when necessary and sanitize it.

Your report must contain:

```text
LAB: JUN-054
TARGET:
EXPECTED METHOD/PATH:
BASELINE:
RESOLUTION:
FIRST REQUEST:
STATUS:
HEADERS:
BODY SUMMARY:
REDIRECTS:
TIMING:
ERROR CLASSIFICATION:
WHAT IS PROVEN:
WHAT IS NOT PROVEN:
HYPOTHESES:
NEXT TEST:
AUTHORIZED REMEDIATION:
VERIFICATION:
PREVENTION:
LIMITATIONS:
```

### Acceptance Criteria

- [ ] Requests are read-only/authorized.
- [ ] DNS, connection, TLS, and HTTP outcomes are distinguished.
- [ ] Status codes are interpreted as HTTP evidence, not generic network evidence.
- [ ] Timeouts are bounded.
- [ ] Sensitive verbose output is not published.
- [ ] Redirects are identified explicitly.
- [ ] Final verification repeats the original endpoint operation.

# 28. Cleanup

Stop the Python HTTP server with `Ctrl+C`, verify port 8080 is no longer listening, then remove only your lab workspace:

```bash
ss -ltn 'sport = :8080'
rm -rf /tmp/veriqta-jun-054
```

Before the `rm`, verify the path is exactly your controlled lab directory.

---

# Knowledge Check

### 1. What does curl test that ping does not?

**Answer:** It can test application-layer protocols such as HTTP, including request/response behavior.

### 2. What is the difference between a 404 and connection refused?

**Answer:** 404 is an HTTP response; refusal occurs before an HTTP response is obtained.

### 3. What does `-I` do?

**Answer:** Requests headers using HEAD where supported.

### 4. What does `-i` do?

**Answer:** Includes response headers with normal response output.

### 5. Why use `-sS`?

**Answer:** It suppresses progress while preserving error messages.

### 6. What does `-o /dev/null` do?

**Answer:** Discards the response body.

### 7. What does `-w` do?

**Answer:** Prints selected transfer metadata such as status or timing.

### 8. What does `-L` do?

**Answer:** Follows redirects.

### 9. Why can redirects be security-sensitive?

**Answer:** Credentials/headers may cross boundaries depending on behavior; inspect destinations and authorization.

### 10. What does a 503 prove?

**Answer:** An HTTP endpoint responded with Service Unavailable; it does not by itself identify root cause.

### 11. What does connection refused suggest?

**Answer:** The connection was actively rejected/no accepting listener at that path, but more evidence is needed.

### 12. What does a timeout prove?

**Answer:** The operation did not complete within the configured time.

### 13. Why bound curl timeouts?

**Answer:** To make diagnostics predictable and prevent hanging checks.

### 14. Should `-k` be a default fix?

**Answer:** No. It disables certificate verification and can hide a real trust/name problem.

### 15. Why inspect DNS first?

**Answer:** A name-resolution failure occurs before connecting to the HTTP endpoint.

### 16. Can HEAD differ from GET?

**Answer:** Yes; servers/apps can implement them differently.

### 17. Why measure timing components?

**Answer:** They help locate where time is spent but require correlation before root-cause claims.

### 18. What should verbose output be treated as?

**Answer:** Potentially sensitive diagnostic evidence that must be sanitized.

### 19. What should recovery verification repeat?

**Answer:** The exact method/path and original health/user operation.

### 20. What is the core layered model?

**Answer:** Resolve → connect → TLS if applicable → HTTP request → status/headers/body → application behavior.

---

# Interview Preparation

### 1. How would you investigate an HTTP health-check failure?

**Worked answer:** Resolve the name, run a bounded request, capture status/headers, classify connection/TLS/HTTP behavior, inspect timing, compare expected path/method, form hypotheses, and verify the original health check after remediation.

### 2. Is HTTP 500 a network failure?

**Worked answer:** No. It is an HTTP server error response; transport worked far enough to receive it.

### 3. Why use curl instead of a browser?

**Worked answer:** curl provides reproducible, scriptable request/response evidence and exposes protocol details cleanly.

### 4. How do you distinguish DNS from connection failure?

**Worked answer:** Check resolution separately and classify curl errors before interpreting HTTP.

### 5. What does `curl -v` add?

**Worked answer:** Detailed resolution/connection/TLS/request/response diagnostics, with a need to sanitize secrets.

### 6. Why not always follow redirects?

**Worked answer:** A redirect may itself be the unexpected behavior, and following it can hide the first response.

### 7. How do you test latency?

**Worked answer:** Use curl write-out timings across multiple bounded samples and correlate with server/network evidence.

### 8. How do you test a production endpoint safely?

**Worked answer:** Use authorized read-only methods/paths, bounded timeouts, no credential leakage, and no side-effecting methods without approval.

### 9. What is wrong with declaring success from HTTP 200 alone?

**Worked answer:** It may be a generic page or wrong endpoint; validate expected content/semantics and original operation.

### 10. What belongs in an HTTP incident report?

**Worked answer:** Target, resolution, method/path, status, headers, timing, errors, evidence, hypotheses, remediation, verification, prevention, limitations.

---

# Portfolio Evidence

Create a sanitized artifact named `jun-054-engineering-report.md`. Include the problem statement, baseline, commands selected, important observations, reasoning, failure evidence, recovery or remediation design, verification, and prevention. Remove internal hostnames, addresses, usernames, keys, tokens, and other sensitive infrastructure details before publishing.

**Possible portfolio statement:**

> Completed a production-style Linux investigation using evidence-driven troubleshooting, controlled testing, explicit verification, and documented remediation/prevention rather than command guessing.

---

# Completion Checklist

- [ ] I established a baseline before changing state.
- [ ] I can explain the primary tool and its important options.
- [ ] I separated observations from conclusions.
- [ ] I completed the guided experiments.
- [ ] I investigated controlled failures rather than guessing.
- [ ] I completed the production incident.
- [ ] I considered multiple hypotheses where appropriate.
- [ ] I returned verification to the original symptom.
- [ ] I documented prevention, not only recovery.
- [ ] I completed the independent challenge and report.
- [ ] I sanitized any portfolio evidence.
- [ ] I completed cleanup or verified that no cleanup was required.

---

# Engineering Mental Model

```text
SYMPTOM
   ↓
BASELINE
   ↓
QUESTION
   ↓
TEST
   ↓
EVIDENCE
   ↓
INTERPRETATION
   ↓
HYPOTHESES
   ↓
DISCRIMINATING TEST
   ↓
ROOT CAUSE
   ↓
AUTHORIZED REMEDIATION
   ↓
VERIFY ORIGINAL OPERATION
   ↓
PREVENT RECURRENCE
```

A command is useful only when you know what question it answers and what its result does—and does not—prove.

---

## Next Lab — JUN-055 — SSH Fundamentals

Continue to the next lab only after you can explain the evidence you collected here without relying on memorized command sequences.
