# JUN-055 — SSH Fundamentals

**VERIQTA | LINUX WORLD — HANDS-ON ENGINEERING LAB**

**Suggested repository path:** `Junior/03-Networking/JUN-055-SSH-Fundamentals/README.md`

| Field | Lab Standard |
|---|---|
| Lab ID | JUN-055 |
| Track | Junior Engineer |
| Topic | safe remote shell access, SSH client behavior, host identity, sessions, and first-line troubleshooting |
| Difficulty | 3/5 |
| Estimated Duration | 240–270 minutes |
| Operating System | Ubuntu Server 24.04 LTS |
| Required Privileges | Standard user; `sudo` only where explicitly justified |
| Primary Tools | `ssh`, `ssh-keygen`, `ss`, `getent`, `systemctl` |
| Lab Style | Guided follow-along engineering investigation |
| Final Deliverable | SSH Access Investigation Report |
| Next Lab | JUN-056 — SSH Keys |

> **Evidence rule:** Outputs shown in this lab are labeled illustrative, expected, or supplied incident evidence. Your own report must contain what you actually observed. If something is absent, record **Not observed** rather than inventing evidence.

---
# 1. Production Mission

You have been asked to connect from an administration host to `app-admin-01` using SSH. The team reports that some engineers can connect while others see host-verification, timeout, or authentication errors. Your job is to understand the SSH connection lifecycle, establish safe client-side evidence, create a controlled localhost SSH test when available, and troubleshoot without weakening security or locking yourself out.

This lab teaches SSH fundamentals. Key creation and key lifecycle are reserved for JUN-056; configuration editing is reserved for JUN-057.

# 2. Workflow

```text
VERIFY CLIENT → IDENTIFY TARGET → RESOLVE → ROUTE → PORT/LISTENER CONTEXT
→ FIRST SSH CONNECTION → HOST KEY PROMPT → USER AUTHENTICATION → SESSION
→ CLIENT VERBOSITY → CONTROLLED FAILURES → INCIDENT → REFERENCE
→ DECISION TREE → FINAL CHALLENGE
```

# 3. Objectives

You will distinguish SSH client/server roles, understand TCP port 22 as a default rather than a guarantee, explain host keys versus user authentication, understand `known_hosts`, connect with explicit users/hosts/ports, use bounded connection timeouts, use `-v` diagnostics, distinguish transport from host verification and authentication failures, and preserve existing access while troubleshooting.

# 4. SSH Safety Rules

- Never close your only working remote session while testing changes.
- Do not delete `known_hosts` wholesale to silence a warning.
- Do not bypass host-key checking as a routine fix.
- Do not change `sshd_config` in this lab.
- Do not restart SSH in this lab.
- Do not publish usernames, internal hostnames, addresses, or host-key fingerprints without authorization.
- Do not test credentials against systems you do not own or administer.

# 5. Verify Tools

```bash
hostname
whoami
command -v ssh
ssh -V
command -v getent
command -v ss
```

`ssh -V` may print to standard error; that is normal.

# 6. Understand the SSH Roles

```text
SSH CLIENT                           SSH SERVER
ssh                                  sshd
 |                                    |
 |---- TCP connection --------------->|
 |<--- server host identity ----------|
 |---- user authentication ---------->|
 |<--- encrypted session ------------>|
```

The server host key proves server identity to the client when correctly verified. It is not the same as your user authentication credential.

# 7. Inspect Client Configuration Sources Without Editing

```bash
ls -la ~/.ssh 2>/dev/null || true
```

Do not publish the contents of private keys. If `~/.ssh` does not exist, record that.

Inspect whether a per-user client config exists:

```bash
test -f ~/.ssh/config && echo 'user SSH config exists' || echo 'no user SSH config observed'
```

System-wide client configuration commonly lives under `/etc/ssh/ssh_config` and `/etc/ssh/ssh_config.d/`. This lab does not modify it.

# 8. Determine Effective Client Settings

For a harmless target name, use:

```bash
ssh -G example.com | head -n 30
```

`-G` prints evaluated client configuration and exits without connecting. This is valuable when aliases/configuration change user, port, identity files, proxy behavior, or host-key policy.

Record the effective `hostname`, `user`, and `port` fields.

# 9. Establish Network Context

For an approved target:

```bash
getent ahostsv4 YOUR_SSH_HOST
```

Then inspect route selection for the resolved address:

```bash
ip route get YOUR_TARGET_IP
```

Do not assume SSH is the problem if the name does not resolve or no route exists.

# 10. First SSH Syntax

The standard form is:

```bash
ssh USER@HOST
```

A non-default port is specified with uppercase `-p`:

```bash
ssh -p 2222 USER@HOST
```

Do not confuse this with lowercase options from other tools.

# 11. Bound Connection Establishment

For troubleshooting:

```bash
ssh -o ConnectTimeout=5 USER@HOST
```

`-o` supplies a client configuration option on the command line. A bounded timeout avoids waiting indefinitely.

# 12. Host-Key Verification

On first contact, SSH may present a host-key fingerprint and ask whether you trust the host. Do not type `yes` merely because the prompt appears. Verify the fingerprint through an approved source or administrator.

A changed-host-key warning is security-significant. It can result from legitimate rebuilds, DNS/IP reuse, or a security problem. Investigate before altering `known_hosts`.

# 13. known_hosts Fundamentals

The user host-key database is commonly:

```text
~/.ssh/known_hosts
```

Inspect metadata, not secrets:

```bash
ls -l ~/.ssh/known_hosts 2>/dev/null || true
```

To remove a specific stale entry **only after the replacement host identity has been independently verified**, use a targeted command such as:

```bash
ssh-keygen -R HOSTNAME
```

Do not run this just to suppress a warning.

# 14. Authentication Is a Separate Stage

A successful TCP connection and verified server host key do not mean user authentication will succeed. Depending on server policy, SSH may use public keys, passwords, certificates, MFA, or other methods.

This lab observes authentication behavior; JUN-056 teaches SSH keys in depth.

# 15. Use Verbose Diagnostics

```bash
ssh -v -o ConnectTimeout=5 USER@HOST
```

`-v` enables debug output. `-vv` and `-vvv` increase detail. Start with the least verbosity needed.

Verbose logs can reveal usernames, hostnames, addresses, identity-file paths, algorithms, and authentication attempts. Sanitize before sharing.

# 16. Read the Connection Stages

In verbose output, identify evidence for:

1. configuration evaluation;
2. name/address selection;
3. TCP connection;
4. protocol negotiation;
5. host-key verification;
6. authentication methods;
7. session establishment.

The stage where progress stops matters more than memorizing every debug line.

# 17. Controlled Localhost Test

First check whether an SSH server is active locally:

```bash
systemctl is-active ssh 2>/dev/null || true
ss -ltn 'sport = :22'
```

If no local SSH server is available, do not install or start one merely for this lab. Record the limitation and use an authorized lab VM/target.

If a local server is already available and you are authorized to use it:

```bash
ssh -o ConnectTimeout=5 localhost
```

Do not change authentication configuration. Observe which stage succeeds or fails.

# 18. Failure Lab — Name Resolution

```text
ssh: Could not resolve hostname app-admn-01: Temporary failure in name resolution
```

First verify the exact spelling and resolver evidence. Do not change SSH configuration to solve a DNS failure.

# 19. Failure Lab — Connection Refused

Supplied evidence:

```text
ssh: connect to host 10.20.30.40 port 22: Connection refused
```

This is transport-stage evidence. Plausible causes include no listener on that address/port or an active rejection. Do not diagnose authentication yet because authentication was never reached.

# 20. Failure Lab — Timeout

```text
ssh: connect to host 10.20.30.40 port 22: Connection timed out
```

Investigate route, filtering, target availability, correct port, and network path. A timeout is different from refusal.

# 21. Failure Lab — Host-Key Changed

Supplied warning:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Correct response: stop, verify whether the host was legitimately rebuilt/replaced, obtain the approved new fingerprint through a trusted channel, then update only the specific stale entry if authorized. Never teach `StrictHostKeyChecking=no` as the fix.

# 22. Failure Lab — Permission Denied

```text
Permission denied (publickey).
```

Transport and host verification may have succeeded; authentication failed. Confirm intended username, server policy, offered credentials, and authorization. JUN-056 will go deeper into key authentication.

# 23. Production Incident — Engineers Cannot Reach New Admin Host

**Target:** `app-admin-01.internal.example`

**Expected:** engineers connect as `opsuser` on TCP 22.

### Evidence 1 — resolution

```text
10.60.10.44 STREAM app-admin-01.internal.example
```

### Evidence 2 — one engineer's verbose SSH

```text
Connecting to app-admin-01.internal.example [10.60.10.44] port 22.
connect to address 10.60.10.44 port 22: Connection refused
```

### Evidence 3 — server-side socket evidence supplied by platform team

```text
LISTEN ... 10.60.10.44:2222 ... users:(("sshd",pid=...))
```

### Hypotheses

1. SSH service moved to 2222 intentionally.
2. Service is misconfigured and should still use 22.
3. A deployment changed listener configuration unexpectedly.

### Evidence 4 — approved service inventory

```text
app-admin-01 SSH port: 2222
change ticket: approved migration
```

Root cause: client connection expectation was stale; the server is intentionally on port 2222.

### Safe connection

```bash
ssh -p 2222 -o ConnectTimeout=5 opsuser@app-admin-01.internal.example
```

Verify the server host fingerprint through the approved source before accepting it.

### Prevention

Update inventory/runbooks/client configuration through controlled processes; include port in onboarding documentation; monitor SSH listener state.



# Extended Follow-Along Engineering Practice — SSH Connection Lifecycle

This block deepens the same lab scope. Work through it in order; do not treat it as optional reading. For every command, record your own result and write one sentence stating what the evidence proves and one sentence stating what it does not prove.

## Practice 1 — Inspect effective destination

Client config can silently change the destination/user/port.

Run the following only on the authorized lab host/target:

```bash
ssh -G YOUR_AUTHORIZED_HOST | grep -E '^(hostname|user|port) '
```

Do this before blaming the server. Effective client state is evidence.

Record your evidence:

```text
HOSTNAME:
USER:
PORT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 2 — Resolve target explicitly

SSH hostname errors are not authentication errors.

Run the following only on the authorized lab host/target:

```bash
getent ahostsv4 YOUR_AUTHORIZED_HOST
```

Record the address selected for later route comparison.

Record your evidence:

```text
RESOLVED ADDRESS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 3 — Inspect route to SSH target

Know the local path decision before interpreting timeouts.

Run the following only on the authorized lab host/target:

```bash
ip route get YOUR_TARGET_IP
```

A route selection is not proof the remote port is reachable.

Record your evidence:

```text
ROUTE:
SOURCE ADDRESS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 4 — Use bounded connection attempt

Avoid indefinite waits.

Run the following only on the authorized lab host/target:

```bash
ssh -v -o ConnectTimeout=5 USER@HOST
```

Identify the last successful SSH stage and first failure.

Record your evidence:

```text
LAST SUCCESSFUL STAGE:
FIRST FAILURE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 5 — Compare default and explicit port

Port assumptions are common incident causes.

Run the following only on the authorized lab host/target:

```bash
ssh -G HOST | grep '^port '
ssh -p EXPECTED_PORT -o ConnectTimeout=5 USER@HOST
```

Do not test arbitrary ports; use approved inventory.

Record your evidence:

```text
EFFECTIVE PORT:
APPROVED PORT:
RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 6 — Inspect known-host metadata

Trust state matters before authentication.

Run the following only on the authorized lab host/target:

```bash
ls -l ~/.ssh/known_hosts 2>/dev/null || true
ssh-keygen -F HOST 2>/dev/null || true
```

`ssh-keygen -F` can find entries without deleting them.

Record your evidence:

```text
ENTRY OBSERVED?:
FINGERPRINT/KEY INFO IF SAFE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 7 — Distinguish host identity from user auth

Verbose logs expose separate phases.

Run the following only on the authorized lab host/target:

```bash
ssh -v -o ConnectTimeout=5 USER@HOST
```

Locate host-key verification and authentication-method lines separately.

Record your evidence:

```text
HOST IDENTITY STAGE:
USER AUTH STAGE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 8 — Test command execution after login

A session can connect but a requested command may still fail.

Run the following only on the authorized lab host/target:

```bash
ssh USER@HOST 'whoami; hostname'
```

Only use on an authorized target. Record remote identity and host.

Record your evidence:

```text
REMOTE USER:
REMOTE HOST:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 9 — Observe exit status

Automation depends on SSH command exit behavior.

Run the following only on the authorized lab host/target:

```bash
ssh USER@HOST 'true'
echo $?
```

An exit status of zero here means the remote command succeeded through SSH; it does not validate every remote operation.

Record your evidence:

```text
EXIT STATUS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 10 — Classify a permission-denied failure

Authentication failures occur after earlier stages.

Run the following only on the authorized lab host/target:

```bash
ssh -v -o ConnectTimeout=5 USER@HOST
```

Record methods offered/accepted without publishing sensitive details.

Record your evidence:

```text
TRANSPORT SUCCEEDED?:
HOST VERIFIED?:
AUTH RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 11 — Protect the recovery session

Before any later SSH configuration work, inventory current sessions.

Run the following only on the authorized lab host/target:

```bash
who
ss -tn | grep ':22' || true
```

Do not kill sessions. The goal is to recognize which connection is preserving access.

Record your evidence:

```text
CURRENT SESSION OBSERVED?:
RECOVERY ACCESS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 12 — Create an SSH incident timeline

Timestamp evidence so logs and monitoring can be correlated.

Run the following only on the authorized lab host/target:

```bash
date -Is
ssh -v -o ConnectTimeout=5 USER@HOST
```

Preserve the exact error and stage with the timestamp.

Record your evidence:

```text
TIME:
TARGET:
ERROR/STAGE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.




# Engineering Scenario Drill Bank

Use these as short incident rehearsals. For each scenario, do **not** jump directly to a fix. Write: symptom, expected state, first evidence to collect, two plausible hypotheses, one discriminating test for each hypothesis, safe remediation owner, verification of the original operation, and one prevention control.

## Drill 1 — Hostname Typo

**Scenario:** The production symptom is **hostname typo**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: hostname typo
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

## Drill 2 — Wrong Destination Port

**Scenario:** The production symptom is **wrong destination port**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: wrong destination port
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

## Drill 3 — Connection Refused

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

## Drill 4 — Connection Timeout

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

## Drill 5 — Changed Host Key

**Scenario:** The production symptom is **changed host key**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: changed host key
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

## Drill 6 — Wrong Username

**Scenario:** The production symptom is **wrong username**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: wrong username
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

## Drill 7 — Permission Denied Publickey

**Scenario:** The production symptom is **permission denied publickey**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: permission denied publickey
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

## Drill 8 — Client Alias Points To Old Host

**Scenario:** The production symptom is **client alias points to old host**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: client alias points to old host
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

## Drill 9 — Verbose Log Stops Before Authentication

**Scenario:** The production symptom is **verbose log stops before authentication**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: verbose log stops before authentication
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

## Drill 10 — Session Opens But Remote Command Fails

**Scenario:** The production symptom is **session opens but remote command fails**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: session opens but remote command fails
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


# 24. Command-and-Flag Reference

| Command | Purpose |
|---|---|
| `ssh USER@HOST` | Connect with explicit user |
| `ssh -p PORT USER@HOST` | Connect to non-default port |
| `ssh -v USER@HOST` | First-level debug output |
| `ssh -vv` / `-vvv` | Increasing debug detail |
| `ssh -o ConnectTimeout=5 ...` | Bound connection establishment |
| `ssh -G HOST` | Print effective client configuration without connecting |
| `ssh-keygen -R HOST` | Remove one known-host entry after identity verification |
| `getent ahostsv4 HOST` | Resolve through NSS |
| `ip route get IP` | Inspect route selection |

# 25. SSH Troubleshooting Decision Tree

```text
SSH CONNECTION FAILS
       |
       v
NAME RESOLVES?
 |           |
NO          YES
 |           v
DNS       CORRECT PORT/ROUTE?
             |          |
            NO         YES
             |          v
        correct target TCP CONNECTS?
                        |      |
                       NO     YES
                        |      v
                 refusal/timeout HOST KEY VALID?
                               |       |
                              NO      YES
                               |       v
                        verify identity AUTHENTICATION?
                                       |       |
                                      NO      YES
                                       |       v
                                  user/cred   SESSION/COMMAND
                                  policy
```

# 26. Independent Final Challenge — SSH Access Investigation

Given an authorized host and account, produce a read-only investigation that identifies effective client settings, resolution, route, target port, connection stage, host-key state, and authentication stage. Do not edit server configuration or remove host-key entries unless the challenge explicitly supplies independent identity verification and authorization.

Report template:

```text
LAB: JUN-055
CLIENT HOST:
TARGET HOST:
EXPECTED USER:
EXPECTED PORT:
RESOLUTION:
ROUTE:
EFFECTIVE SSH CONFIG:
TCP CONNECTION STAGE:
HOST-KEY VERIFICATION:
AUTHENTICATION STAGE:
SESSION RESULT:
VERBOSE EVIDENCE SUMMARY:
WHAT IS PROVEN:
WHAT IS NOT PROVEN:
HYPOTHESES:
NEXT TEST:
AUTHORIZED OWNER:
VERIFICATION:
PREVENTION:
LIMITATIONS:
```

### Acceptance Criteria

- [ ] Client and server roles are distinguished.
- [ ] Host identity and user authentication are distinguished.
- [ ] The existing SSH session is protected.
- [ ] No host-key warning is bypassed.
- [ ] No server configuration is changed.
- [ ] Verbose evidence is sanitized.
- [ ] Failures are classified by connection stage.
- [ ] The exact next test follows from evidence.

# 27. Cleanup

No persistent change is required. Close only test SSH sessions you opened. Do not close the administrative session that provides your only access. No SSH service restart is required.

---

# Knowledge Check

### 1. What is the SSH client program?

**Answer:** `ssh`.

### 2. What is the SSH server daemon commonly called?

**Answer:** `sshd`.

### 3. What is SSH default TCP port?

**Answer:** 22, though servers can be configured differently.

### 4. What does `-p` specify?

**Answer:** The destination SSH port.

### 5. What does `-v` do?

**Answer:** Enables client debug output.

### 6. What does `ssh -G` do?

**Answer:** Prints evaluated client configuration without connecting.

### 7. What is a host key for?

**Answer:** Authenticating the server host to the client.

### 8. Is a host key the same as a user key?

**Answer:** No.

### 9. What is `known_hosts` for?

**Answer:** Recording known server host identities/fingerprints for future verification.

### 10. Should you delete known_hosts after a warning?

**Answer:** No; verify identity and update only the affected entry if authorized.

### 11. What does connection refused indicate?

**Answer:** TCP connection was actively rejected/no accepting service at that path; authentication was not reached.

### 12. What does timeout indicate?

**Answer:** Connection did not complete within the allowed time; investigate path/filter/target/port.

### 13. What does permission denied indicate?

**Answer:** Authentication failed after reaching that stage.

### 14. Why preserve a working SSH session?

**Answer:** It provides recovery access if a test or later configuration change fails.

### 15. Why sanitize verbose logs?

**Answer:** They may reveal sensitive infrastructure/user/credential-path information.

### 16. Does DNS success prove SSH works?

**Answer:** No.

### 17. Does a TCP connection prove authentication works?

**Answer:** No.

### 18. Why verify host fingerprints independently?

**Answer:** To avoid trusting an impostor or unintended replacement.

### 19. What is the correct response to a changed host key?

**Answer:** Stop and verify the host identity/change before updating trust.

### 20. What is the layered SSH model?

**Answer:** Resolution → route/port → TCP → protocol/host identity → user authentication → session.

---

# Interview Preparation

### 1. How do you troubleshoot SSH connection failures?

**Worked answer:** Classify the stage: resolution, route/port, TCP, host-key verification, authentication, or session. Gather evidence at that stage and avoid changing unrelated configuration.

### 2. What is the difference between timeout and permission denied?

**Worked answer:** Timeout occurs before authentication completes; permission denied is an authentication-stage result.

### 3. Why is StrictHostKeyChecking=no dangerous as a generic fix?

**Worked answer:** It weakens server identity verification and can hide man-in-the-middle or unintended-host risks.

### 4. How do you handle a changed host key?

**Worked answer:** Verify the rebuild/replacement and new fingerprint through a trusted source, then update only the specific stale entry if authorized.

### 5. What does `ssh -G` help diagnose?

**Worked answer:** Unexpected effective user, hostname, port, identity, proxy, and other client settings.

### 6. Why might SSH use a port other than 22?

**Worked answer:** Administrative policy or architecture may choose another port; inventory/config must define the expected state.

### 7. What would you capture before changing SSH config?

**Worked answer:** Working access, effective config, current listener/session state, backup, validation plan, rollback path.

### 8. What does successful SSH transport not prove?

**Worked answer:** It does not prove host identity was accepted, user authentication succeeded, or the requested session/command works.

### 9. How do you use verbose SSH safely?

**Worked answer:** Start with `-v`, increase only if needed, and sanitize sensitive output.

### 10. What is your first priority during SSH troubleshooting on a remote host?

**Worked answer:** Preserve reliable access and avoid lockout while collecting evidence.

---

# Portfolio Evidence

Create a sanitized artifact named `jun-055-engineering-report.md`. Include the problem statement, baseline, commands selected, important observations, reasoning, failure evidence, recovery or remediation design, verification, and prevention. Remove internal hostnames, addresses, usernames, keys, tokens, and other sensitive infrastructure details before publishing.

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

## Next Lab — JUN-056 — SSH Keys

Continue to the next lab only after you can explain the evidence you collected here without relying on memorized command sequences.
