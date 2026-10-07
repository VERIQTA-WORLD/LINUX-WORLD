# JUN-053 — ping and Connectivity Testing

**VERIQTA | LINUX WORLD — HANDS-ON ENGINEERING LAB**

**Suggested repository path:** `Junior/03-Networking/JUN-053-ping-and-Connectivity-Testing/README.md`

| Field | Lab Standard |
|---|---|
| Lab ID | JUN-053 |
| Track | Junior Engineer |
| Topic | ICMP reachability, packet loss, latency, and safe connectivity testing |
| Difficulty | 3/5 |
| Estimated Duration | 210–240 minutes |
| Operating System | Ubuntu Server 24.04 LTS |
| Required Privileges | Standard user; `sudo` only where explicitly justified |
| Primary Tools | `ping`, `ip`, `getent` |
| Lab Style | Guided follow-along engineering investigation |
| Final Deliverable | Network Connectivity Investigation Report |
| Next Lab | JUN-054 — curl and HTTP Testing |

> **Evidence rule:** Outputs shown in this lab are labeled illustrative, expected, or supplied incident evidence. Your own report must contain what you actually observed. If something is absent, record **Not observed** rather than inventing evidence.

---
# 1. Production Mission

Monitoring reports intermittent connectivity between `app-edge-01` and an internal dependency. DNS resolves the dependency name, the local interface is up, and the route table appears plausible. Your task is not to prove that “the network is good.” Your task is to use `ping` correctly, collect ICMP evidence, interpret loss and round-trip time, recognize the limits of that evidence, and decide what should be tested next.

You are working through an active SSH session. This lab is read-only with respect to network configuration. Do **not** change addresses, routes, firewall rules, resolver settings, or interface state.

## What you will build

```text
VERIFY TOOLS → RECORD HOST BASELINE → TEST LOOPBACK → TEST LOCAL ADDRESS
→ TEST GATEWAY → TEST NAME/IP → CONTROL COUNT/TIMEOUT → INTERPRET RTT/LOSS
→ COMPARE DNS VS IP → OBSERVE SAFE FAILURES → PRODUCTION INCIDENT
→ REFERENCE → DECISION TREE → INDEPENDENT CHALLENGE → REPORT
```

# 2. Learning Objectives

By the end you should be able to explain ICMP Echo Request/Reply, run bounded `ping` tests, interpret transmitted/received/loss and RTT statistics, distinguish DNS failure from IP reachability failure, recognize unreachable/timeout behavior, understand why ICMP can be filtered, choose useful targets, avoid endless tests, and state precisely what successful or failed `ping` proves.

# 3. Safety Boundary

`ping` is normally non-destructive, but testing still needs judgment. Query only systems you own or are authorized to test. Avoid flood modes and aggressive intervals. Never “fix” failed ping by disabling a firewall. Do not change the live default route or interface. Preserve your SSH session.

# 4. Verify the Environment

Run:

```bash
hostname
whoami
pwd
command -v ping
command -v ip
command -v getent
```

Record:

```text
HOSTNAME:
USER:
PING PATH:
IP PATH:
GETENT PATH:
```

If `ping` is unavailable, document that limitation rather than installing packages blindly on a managed host.

## Checkpoint 1

You should know which host you are on, which user you are operating as, and whether the required tools exist.

# 5. Establish the Network Baseline

Before sending traffic, inspect local state.

```bash
ip -br addr
ip -4 route show
ip -6 route show
```

`ip -br addr` gives a compact interface/address view. `ip -4 route show` and `ip -6 route show` show the kernel routing tables for each address family.

Record the active interface, one local address, the IPv4 default gateway if present, and whether IPv6 routes exist.

```text
ACTIVE INTERFACE:
LOCAL IPv4:
DEFAULT GATEWAY:
IPv6 PRESENT?:
```

This baseline does not prove remote reachability. It tells you the local starting state.

# 6. Understand What ping Actually Tests

`ping` sends ICMP Echo Request messages and waits for Echo Replies. A reply can provide evidence that a path exists for ICMP between the source and destination. It does **not** prove that SSH, HTTP, a database, or another application is healthy.

```text
PING SUCCEEDS
     ≠
APPLICATION WORKS
```

Likewise:

```text
PING FAILS
     ≠
HOST DEFINITELY DOWN
```

A firewall or policy may drop ICMP while allowing required application traffic.

# 7. First Controlled Test — Loopback

Run a bounded test:

```bash
ping -c 4 127.0.0.1
```

`-c 4` stops after four Echo Requests. This is preferable to an unbounded test in documentation and troubleshooting.

Illustrative output:

```text
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = 0.030/0.041/0.055/0.010 ms
```

Interpret the summary: transmitted is how many requests were sent; received is how many replies returned; packet loss is the missing percentage; RTT statistics summarize observed round-trip delay.

Record your actual result.

## Checkpoint 2

If loopback responds, you have evidence that the local IP/ICMP path is functioning. You have not tested the NIC, gateway, remote network, DNS, or an application.

# 8. Test the Host's Own Address

Find your IPv4 address from the baseline, then test it with four packets. Replace the placeholder deliberately:

```bash
ping -c 4 YOUR_LOCAL_IPV4
```

Record whether it differs from loopback. Explain what both tests do and do not establish.

# 9. Test the Default Gateway

If your host has a default gateway, verify the route selection first:

```bash
ip -4 route show default
```

Then run:

```bash
ping -c 4 YOUR_DEFAULT_GATEWAY
```

Do not assume a failed reply means the gateway is unusable. Some devices restrict ICMP. Record the result as evidence, not as a verdict.

# 10. Use Numeric and Named Targets Deliberately

A hostname test involves name resolution before ICMP can be sent. First inspect resolution:

```bash
getent ahostsv4 example.com
```

Then:

```bash
ping -c 4 example.com
```

Now note the numeric address shown by `ping`. If appropriate in your environment, test that numeric address separately.

The comparison matters:

```text
NAME FAILS + IP WORKS → investigate naming/resolution
NAME WORKS + ICMP FAILS → resolution succeeded; ICMP path/reply did not
NAME WORKS + ICMP WORKS → evidence for resolution + ICMP reachability only
```

# 11. Control the Address Family

Use:

```bash
ping -4 -c 4 example.com
ping -6 -c 4 example.com
```

`-4` forces IPv4. `-6` forces IPv6. IPv6 may be unavailable in your environment; that is a valid observation.

Record each result separately. Do not report “networking failed” because one address family is not configured.

# 12. Control Time and Count

A troubleshooting command should terminate predictably.

```bash
ping -c 5 -W 2 example.com
```

`-W 2` limits the wait for each reply to two seconds on GNU/Linux `ping`. Use modest values. Your environment may produce different timing.

Another useful bounded form is:

```bash
ping -c 10 -i 0.5 example.com
```

`-i 0.5` requests a half-second interval. Do not use aggressive intervals against systems you do not control.

# 13. Read Per-Packet Output

A reply line may resemble:

```text
64 bytes from 203.0.113.20: icmp_seq=1 ttl=54 time=22.4 ms
```

Interpret:

- `icmp_seq` identifies the request/reply sequence.
- `ttl` is the received packet's remaining Time To Live/Hop Limit-style value; do not use it as a precise hop-count measurement without knowing the sender's initial TTL.
- `time` is the measured round-trip time for that reply.

Do not overdiagnose one slow packet. Look at the pattern.

# 14. Read the Summary Correctly

Illustrative summary:

```text
10 packets transmitted, 9 received, 10% packet loss, time 4510ms
rtt min/avg/max/mdev = 18.201/24.552/47.991/8.912 ms
```

This establishes observed loss during this sample. It does not tell you automatically where the loss occurred or whether the application experienced the same failure.

Record:

```text
TARGET:
PACKETS SENT:
PACKETS RECEIVED:
LOSS:
MIN RTT:
AVG RTT:
MAX RTT:
MDEV:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
```

# 15. Controlled Comparison — Short Sample vs Longer Sample

Run:

```bash
ping -c 4 example.com
ping -c 20 example.com
```

Compare the summaries. A four-packet sample can miss intermittent behavior. A longer sample gives more observations but still represents only the test window.

Do not turn this into indefinite monitoring.

# 16. Recognize Common Failure Messages

Possible outcomes include no replies, `Destination Host Unreachable`, `Network is unreachable`, or name-resolution errors. These are not interchangeable.

- `Network is unreachable` often points to a local routing/address-family problem before packets can be sent.
- `Destination Host Unreachable` may be generated by a local or intermediate system that cannot reach the destination.
- silence/timeouts mean replies were not observed; they do not identify the reason.
- `Name or service not known`/resolution errors occur before an IP ping can proceed.

# 17. Failure Lab 1 — Name Failure vs Reachability

Use the reserved invalid domain:

```bash
getent hosts does-not-exist.invalid
ping -c 2 does-not-exist.invalid
```

Expected behavior: name resolution should fail. Record the exact error.

Now compare with a known numeric target that is appropriate and authorized in your environment. The lesson is to separate **name lookup** from **IP reachability**.

Worksheet:

```text
SYMPTOM:
NAME LOOKUP RESULT:
NUMERIC TEST RESULT:
WHAT IS KNOWN:
WHAT IS NOT KNOWN:
NEXT TEST:
```

# 18. Failure Lab 2 — Route Selection Before ping

Choose a documentation address such as `192.0.2.123` only for route inspection, not as an assumption that a remote host exists:

```bash
ip route get 192.0.2.123
```

Then, if your environment policy allows, run a short bounded ping:

```bash
ping -c 2 -W 1 192.0.2.123
```

A timeout is expected in many environments but is not guaranteed. The important lesson is that `ip route get` answers **how the kernel would route**, while `ping` asks whether ICMP replies are observed.

# 19. Failure Lab 3 — ICMP Failure Is Not Application Proof

Assume supplied evidence:

```text
ping -c 4 api.internal.example
→ 100% packet loss

curl application health check
→ HTTP 200 OK
```

What conclusion is justified?

Correct reasoning: ICMP replies were not observed, but the HTTP service was reachable. ICMP may be filtered or deprioritized. Do not declare the host down.

# 20. Failure Lab 4 — Intermittent Loss

Supplied evidence:

```text
50 packets transmitted, 47 received, 6% packet loss
rtt min/avg/max/mdev = 2.9/18.4/162.7/31.2 ms
```

Do not jump directly to “bad network.” Form at least three hypotheses: congestion/path instability, host load affecting responses, ICMP rate limiting, or another plausible cause. State what additional evidence would discriminate among them.

# 21. Production Incident — Intermittent Dependency Reachability

**Environment:** `app-edge-01` must reach `inventory-api.internal.example`.

**Impact:** users report intermittent checkout failures.

**Expected state:** DNS should resolve the dependency and the network path should be usable.

### Evidence 1 — DNS

Supplied evidence:

```text
$ getent ahostsv4 inventory-api.internal.example
10.30.40.25 STREAM inventory-api.internal.example
10.30.40.25 DGRAM
10.30.40.25 RAW
```

Record what this proves and does not prove.

### Evidence 2 — route selection

```text
$ ip route get 10.30.40.25
10.30.40.25 via 10.20.10.1 dev ens3 src 10.20.10.25
```

Again, route selection does not prove delivery.

### Evidence 3 — ICMP sample

```text
$ ping -c 20 10.30.40.25
20 packets transmitted, 14 received, 30% packet loss
rtt min/avg/max/mdev = 3.1/38.8/211.4/55.2 ms
```

### Your hypotheses

Write at least three plausible hypotheses and a discriminating next test for each. Do not claim root cause yet.

### Evidence 4 — comparison target

Supplied evidence:

```text
$ ping -c 20 10.30.40.26
20 packets transmitted, 20 received, 0% packet loss
rtt min/avg/max/mdev = 3.0/3.5/4.1/0.3 ms
```

Both destinations use the same first-hop gateway.

### Evidence 5 — host-side observation from dependency owner

```text
During the incident window, inventory-api-01 shows CPU saturation and ICMP/application response delays. Network telemetry shows no corresponding packet loss on the shared path.
```

### Root-cause reasoning

The evidence now supports a destination-host resource problem more strongly than a shared network-path failure. `ping` helped characterize symptoms, but it did not independently identify the root cause.

### Authorized remediation

The application/platform owner follows the approved resource remediation/runbook. You do not change routes or firewall rules based only on ping loss.

### Verification

Repeat the original operation after remediation: DNS lookup, bounded reachability sample, and—outside this lab's deep scope—the actual application health check. Recovery must be judged against the original user impact.

### Prevention

Consider host resource alerts, application latency monitoring, dependency health checks, and runbooks that prevent teams from treating ICMP loss as automatic network failure.

# 22. Misconceptions to Eliminate

1. “Ping works, therefore the application works.” False.
2. “Ping fails, therefore the host is down.” False.
3. “Low average RTT means there was no problem.” False if loss or spikes exist.
4. “A hostname ping tests only networking.” False; it also depends on name resolution.
5. “TTL tells me the exact number of hops.” Not without knowing the sender's initial TTL.
6. “One packet is enough.” Usually poor evidence for intermittent issues.
7. “100% loss tells me the firewall is broken.” It tells you replies were not observed.



# Extended Follow-Along Engineering Practice — ICMP Reachability and Evidence Discipline

This block deepens the same lab scope. Work through it in order; do not treat it as optional reading. For every command, record your own result and write one sentence stating what the evidence proves and one sentence stating what it does not prove.

## Practice 1 — Compare loopback names and addresses

A resolver can map `localhost` before ICMP is sent. Compare the name with explicit loopback addresses so you know which layer contributed to the result.

Run the following only on the authorized lab host/target:

```bash
getent hosts localhost
ping -c 4 localhost
ping -4 -c 4 127.0.0.1
```

If the hostname succeeds, record which address family was selected. The explicit IPv4 test removes that selection ambiguity.

Record your evidence:

```text
LOCALHOST RESOLUTION:
PING localhost RESULT:
PING 127.0.0.1 RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 2 — Observe route choice for the gateway

Before interpreting a gateway ping, confirm how the kernel routes that address.

Run the following only on the authorized lab host/target:

```bash
ip -4 route show default
ip route get YOUR_DEFAULT_GATEWAY
```

The route command describes local forwarding intent. A ping reply adds ICMP evidence; neither alone proves an application path.

Record your evidence:

```text
DEFAULT ROUTE:
ROUTE-GET RESULT:
GATEWAY PING RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 3 — Compare packet counts

A tiny sample can miss intermittent behavior. Compare two bounded samples rather than running ping forever.

Run the following only on the authorized lab host/target:

```bash
ping -c 4 YOUR_AUTHORIZED_TARGET
ping -c 20 YOUR_AUTHORIZED_TARGET
```

Compare loss and RTT distribution. A longer sample is more informative for the window but still not long-term monitoring.

Record your evidence:

```text
4-PACKET LOSS/AVG:
20-PACKET LOSS/AVG:
DIFFERENCE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 4 — Compare IPv4 and IPv6

Dual-stack names can behave differently. Test each family explicitly instead of reporting one combined result.

Run the following only on the authorized lab host/target:

```bash
ping -4 -c 4 YOUR_AUTHORIZED_HOSTNAME
ping -6 -c 4 YOUR_AUTHORIZED_HOSTNAME
```

An IPv6 failure on a host without IPv6 service is not evidence that IPv4 networking is broken.

Record your evidence:

```text
IPv4 RESULT:
IPv6 RESULT:
ADDRESS FAMILY AVAILABLE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 5 — Interpret jitter-like variation

Round-trip values can vary even when no packets are lost. Capture the pattern without claiming a cause.

Run the following only on the authorized lab host/target:

```bash
ping -c 20 YOUR_AUTHORIZED_TARGET
```

Compare min, average, max, and mdev. Large spread is evidence of variation, not automatic proof of congestion.

Record your evidence:

```text
MIN:
AVG:
MAX:
MDEV:
LOSS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 6 — Use a comparison target

A second target can help scope a symptom when chosen deliberately.

Run the following only on the authorized lab host/target:

```bash
ping -c 10 TARGET_A
ping -c 10 TARGET_B
```

If both share a path and only one degrades, destination-specific hypotheses become stronger; topology must be known before making that inference.

Record your evidence:

```text
TARGET A SUMMARY:
TARGET B SUMMARY:
WHY THIS COMPARISON IS VALID:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 7 — Separate resolution timing from ICMP

Resolve first, then ping the returned address so a later error is not misclassified.

Run the following only on the authorized lab host/target:

```bash
getent ahostsv4 YOUR_HOSTNAME
ping -c 4 RETURNED_IP
```

If the name lookup fails, ICMP was never attempted by the hostname command. If numeric ping fails, resolution is no longer the immediate blocker.

Record your evidence:

```text
NAME RESULT:
RETURNED IP:
NUMERIC PING:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 8 — Classify unreachable messages

Different errors occur at different points. Preserve exact wording.

Run the following only on the authorized lab host/target:

```bash
ip route get 192.0.2.123
ping -c 2 -W 1 192.0.2.123
```

A route can exist while no reply arrives. If the kernel says no route, that is materially different evidence.

Record your evidence:

```text
ROUTE RESULT:
PING RESULT:
ERROR CLASS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 9 — Document an intermittent sample

Incident notes need timestamps and sample sizes.

Run the following only on the authorized lab host/target:

```bash
date -Is
ping -c 30 YOUR_AUTHORIZED_TARGET
date -Is
```

Record the test window. This helps correlate with monitoring without pretending the sample covers the whole incident.

Record your evidence:

```text
START:
END:
COUNT:
LOSS:
RTT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 10 — Build an evidence ladder

Combine earlier skills without changing state.

Run the following only on the authorized lab host/target:

```bash
getent ahostsv4 YOUR_HOSTNAME
ip route get YOUR_TARGET_IP
ping -c 5 YOUR_TARGET_IP
```

Write one conclusion per layer: name, route selection, ICMP. Do not collapse them into “network good/bad.”

Record your evidence:

```text
NAME EVIDENCE:
ROUTE EVIDENCE:
ICMP EVIDENCE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 11 — Recognize localhost success limits

A local test can succeed while external networking is broken.

Run the following only on the authorized lab host/target:

```bash
ping -c 4 127.0.0.1
ip -br addr
ip -4 route show
```

Loopback validates a local path only. Interface and route evidence are still required for off-host communication.

Record your evidence:

```text
LOOPBACK:
ACTIVE INTERFACE:
DEFAULT ROUTE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 12 — Write a no-change troubleshooting note

Production troubleshooting should make change history explicit.

Run the following only on the authorized lab host/target:

```bash
date -Is
hostname
ip -br addr
```

Record that no addresses, routes, firewall rules, or resolver settings were changed during evidence collection.

Record your evidence:

```text
TIMESTAMP:
HOST:
CONFIGURATION CHANGES MADE: NONE
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.




# Engineering Scenario Drill Bank

Use these as short incident rehearsals. For each scenario, do **not** jump directly to a fix. Write: symptom, expected state, first evidence to collect, two plausible hypotheses, one discriminating test for each hypothesis, safe remediation owner, verification of the original operation, and one prevention control.

## Drill 1 — Hostname Resolves To Unexpected Address

**Scenario:** The production symptom is **hostname resolves to unexpected address**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: hostname resolves to unexpected address
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

## Drill 2 — Loopback Works But Gateway Does Not Reply

**Scenario:** The production symptom is **loopback works but gateway does not reply**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: loopback works but gateway does not reply
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

## Drill 3 — Gateway Replies But Remote Target Does Not

**Scenario:** The production symptom is **gateway replies but remote target does not**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: gateway replies but remote target does not
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

## Drill 4 — Ipv4 Succeeds While Ipv6 Fails

**Scenario:** The production symptom is **IPv4 succeeds while IPv6 fails**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: IPv4 succeeds while IPv6 fails
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

## Drill 5 — One Target Shows Loss While Comparison Target Does Not

**Scenario:** The production symptom is **one target shows loss while comparison target does not**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: one target shows loss while comparison target does not
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

## Drill 6 — Short Sample Is Clean But Longer Sample Shows Intermittent Loss

**Scenario:** The production symptom is **short sample is clean but longer sample shows intermittent loss**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: short sample is clean but longer sample shows intermittent loss
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

## Drill 7 — High Maximum Rtt With Zero Loss

**Scenario:** The production symptom is **high maximum RTT with zero loss**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: high maximum RTT with zero loss
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

## Drill 8 — Numeric Ip Works But Hostname Fails

**Scenario:** The production symptom is **numeric IP works but hostname fails**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: numeric IP works but hostname fails
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


# 23. Command-and-Flag Reference

| Command | Purpose |
|---|---|
| `ping -c 4 TARGET` | Send four Echo Requests and stop |
| `ping -4 -c 4 TARGET` | Force IPv4 |
| `ping -6 -c 4 TARGET` | Force IPv6 |
| `ping -c 5 -W 2 TARGET` | Bound count and per-reply wait |
| `ping -c 10 -i 0.5 TARGET` | Use a controlled interval |
| `ip -br addr` | Compact interface/address baseline |
| `ip -4 route show` | Show IPv4 routes |
| `ip route get ADDRESS` | Show kernel route selection for a destination |
| `getent ahostsv4 NAME` | Resolve IPv4 addresses through NSS |

| Flag | Meaning |
|---|---|
| `-c N` | Stop after N Echo Requests |
| `-4` | IPv4 only |
| `-6` | IPv6 only |
| `-W N` | Reply timeout in seconds on GNU/Linux ping |
| `-i N` | Interval between requests; use responsibly |

# 24. Troubleshooting Decision Tree

```text
CONNECTIVITY SYMPTOM
       |
       v
DOES THE NAME RESOLVE?
   |             |
  NO            YES
   |             |
DNS/NSS path     v
            RECORD TARGET IP
                 |
                 v
           ROUTE SELECTED?
            |          |
           NO         YES
            |          |
       route/local     v
       config path   BOUNDED PING
                      |      |
                   REPLY   NO REPLY
                      |      |
                      v      v
              ICMP evidence  classify error/
              exists only    timeout; test next layer
```

Decision matrix:

| Evidence | Safe interpretation |
|---|---|
| Name fails, numeric IP works | Investigate name resolution |
| `Network is unreachable` | Investigate local addressing/routing |
| Ping succeeds, app fails | Continue to transport/application testing |
| Ping fails, app works | ICMP likely filtered/deprioritized or otherwise not representative |
| Intermittent loss | Gather broader evidence before assigning root cause |

# 25. Independent Final Challenge — Connectivity Investigation

You are given this report:

```text
Host: batch-worker-02
Dependency: reports.internal.example
Symptom: scheduled jobs intermittently report connection timeouts
Constraint: do not modify network configuration or firewall state
```

Perform a structured investigation on an authorized lab target. Your challenge must include:

### Phase A — Baseline
Record hostname, user, interfaces, addresses, and routes.

### Phase B — Resolution
Resolve the target and record the address actually returned.

### Phase C — Route
Show the route the kernel would select.

### Phase D — Reachability
Run bounded ICMP tests. Choose counts and timeouts deliberately.

### Phase E — Comparison
Test at least one appropriate comparison target and explain why you chose it.

### Phase F — Reasoning
State what your evidence proves and does not prove. Form at least three hypotheses if the symptom remains unexplained.

### Phase G — Recovery design
Do not make unauthorized changes. Describe the next layer you would test and who would own remediation.

## Final Deliverable — Network Connectivity Investigation Report

```text
LAB: JUN-053
HOST:
DATE/TIME:
INVESTIGATOR:

1. SYMPTOM
2. IMPACT
3. SAFETY CONSTRAINTS
4. LOCAL BASELINE
5. NAME-RESOLUTION EVIDENCE
6. ROUTE-SELECTION EVIDENCE
7. ICMP TESTS
8. LOSS/RTT INTERPRETATION
9. COMPARISON TARGET
10. WHAT THE EVIDENCE PROVES
11. WHAT IT DOES NOT PROVE
12. HYPOTHESES
13. NEXT TESTS
14. AUTHORIZED REMEDIATION OWNER
15. VERIFICATION PLAN
16. PREVENTION
17. LIMITATIONS
18. CONFIGURATION CHANGES MADE: NONE
19. ENGINEERING CONCLUSION
```

### Acceptance Criteria

- [ ] Tests are bounded and safe.
- [ ] Actual observations are separated from illustrative output.
- [ ] Name resolution and IP reachability are treated separately.
- [ ] Route selection is recorded before conclusions about remote reachability.
- [ ] Packet loss and RTT are interpreted correctly.
- [ ] Successful ping is not treated as application proof.
- [ ] Failed ping is not treated as proof the host is down.
- [ ] At least three hypotheses are recorded where evidence remains ambiguous.
- [ ] No active network configuration is changed.
- [ ] The report identifies a rational next test.

# 26. Cleanup

This lab makes no persistent configuration changes. If you created notes under a temporary directory, remove only the directory you created after verifying its path. No network service restart, route change, firewall change, or cache flush is required.

---

# Knowledge Check

### 1. What protocol does `ping` primarily use?

**Answer:** ICMP Echo Request and Echo Reply for the selected IP family.

### 2. What does `-c 4` do?

**Answer:** It stops the test after four Echo Requests, producing a bounded sample.

### 3. Does successful ping prove an HTTP service is healthy?

**Answer:** No. It proves only that ICMP replies were observed for that path/sample.

### 4. Does 100% packet loss prove the host is powered off?

**Answer:** No. ICMP may be filtered, rate-limited, deprioritized, or lost for other reasons.

### 5. Why test a hostname and numeric address separately?

**Answer:** To separate name-resolution behavior from IP reachability.

### 6. What does `ip route get ADDRESS` tell you?

**Answer:** The route the local kernel would select, including next hop/interface/source where available.

### 7. What does it not prove?

**Answer:** It does not prove packets reach the destination or that the destination responds.

### 8. What is packet loss?

**Answer:** The percentage of sent Echo Requests for which replies were not observed during the sample.

### 9. Why can average RTT be misleading?

**Answer:** It can hide spikes, variation, or loss; inspect min/max/mdev and individual behavior too.

### 10. What does `-4` do?

**Answer:** Forces IPv4.

### 11. What does `-6` do?

**Answer:** Forces IPv6.

### 12. Why use bounded counts?

**Answer:** They make tests predictable, reduce unnecessary traffic, and produce repeatable evidence.

### 13. What might `Network is unreachable` suggest?

**Answer:** A local routing/address-family problem preventing route selection.

### 14. Why is one ping packet weak evidence?

**Answer:** It cannot characterize intermittent loss or latency variation.

### 15. What is a good first response to intermittent loss?

**Answer:** Gather broader evidence and comparison tests before assigning root cause.

### 16. Should you disable a firewall because ping fails?

**Answer:** No. Diagnose policy and required application traffic; ICMP failure alone does not justify disabling controls.

### 17. Why record the exact target IP?

**Answer:** A hostname may resolve to multiple or changing addresses; the actual tested destination matters.

### 18. Can ICMP be treated differently from application traffic?

**Answer:** Yes. Networks and hosts may filter, rate-limit, or deprioritize ICMP.

### 19. What should verification return to?

**Answer:** The original user/service symptom, not merely a successful ping.

### 20. What is the central evidence rule?

**Answer:** State what was observed and what it proves; do not turn a single test into a broader conclusion.

---

# Interview Preparation

### 1. How would you use ping during an outage?

**Worked answer:** I would first identify the exact target, verify resolution and local route selection, run a bounded sample, interpret loss/RTT, compare with another meaningful target, and treat the result as ICMP evidence rather than proof of application health.

### 2. A server does not answer ping. Is it down?

**Worked answer:** Not necessarily. I would consider ICMP filtering/rate limiting and test the required service with an appropriate transport/application tool.

### 3. Why test by IP after a hostname fails?

**Worked answer:** It helps determine whether the failure is in name resolution or in reachability after an address is known.

### 4. What does packet loss tell you?

**Worked answer:** It tells me replies were missing during the sample. It does not by itself locate the loss or identify root cause.

### 5. How do you investigate intermittent latency?

**Worked answer:** Use bounded samples, compare targets, correlate timing with host/network/application metrics, and avoid diagnosing from average RTT alone.

### 6. What is wrong with endless ping in a runbook?

**Worked answer:** It creates uncontrolled traffic and weakens repeatability. I prefer explicit counts and timeouts.

### 7. How do you distinguish route problems from remote silence?

**Worked answer:** Inspect local route selection first; a route error differs from packets being sent with no replies.

### 8. Why is a comparison target useful?

**Worked answer:** It can help determine whether symptoms are shared across a path or specific to one destination, while still requiring caution.

### 9. What should you document?

**Worked answer:** Exact target, resolution, route, command options, sample size, loss, RTT, comparison evidence, limitations, and next test.

### 10. What is the biggest ping troubleshooting mistake?

**Worked answer:** Treating success or failure as a complete network/application diagnosis instead of one piece of evidence.

---

# Portfolio Evidence

Create a sanitized artifact named `jun-053-engineering-report.md`. Include the problem statement, baseline, commands selected, important observations, reasoning, failure evidence, recovery or remediation design, verification, and prevention. Remove internal hostnames, addresses, usernames, keys, tokens, and other sensitive infrastructure details before publishing.

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

## Next Lab — JUN-054 — curl and HTTP Testing

Continue to the next lab only after you can explain the evidence you collected here without relying on memorized command sequences.
