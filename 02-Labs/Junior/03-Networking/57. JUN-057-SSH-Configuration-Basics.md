# JUN-057 — SSH Configuration Basics

**VERIQTA | LINUX WORLD — HANDS-ON ENGINEERING LAB**

**Suggested repository path:** `Junior/03-Networking/JUN-057-SSH-Configuration-Basics/README.md`

| Field | Lab Standard |
|---|---|
| Lab ID | JUN-057 |
| Track | Junior Engineer |
| Topic | SSH client/server configuration basics, effective configuration, validation, safe changes, rollback, and lockout prevention |
| Difficulty | 4/5 |
| Estimated Duration | 300–330 minutes |
| Operating System | Ubuntu Server 24.04 LTS |
| Required Privileges | Standard user; `sudo` only where explicitly justified |
| Primary Tools | `ssh`, `sshd`, `ssh-keygen`, `systemctl`, `ss`, `cp`, `diff` |
| Lab Style | Guided follow-along engineering investigation |
| Final Deliverable | SSH Configuration Change and Verification Report |
| Next Lab | JUN-058 — Bash Fundamentals |

> **Evidence rule:** Outputs shown in this lab are labeled illustrative, expected, or supplied incident evidence. Your own report must contain what you actually observed. If something is absent, record **Not observed** rather than inventing evidence.

---
# 1. Production Mission

A hardened administration host requires a small SSH configuration change. Because SSH is your access path, a careless edit can lock out the team. Your task is to learn where client and server configuration comes from, inspect effective settings, back up before change, validate syntax before reload, test in a second session, keep rollback available, and distinguish configuration intent from runtime state.

# 2. Workflow

```text
PROTECT ACCESS → INVENTORY CURRENT STATE → LOCATE CONFIG SOURCES
→ READ CLIENT CONFIG → READ SERVER CONFIG → EFFECTIVE CONFIG
→ BACKUP → CONTROLLED CHANGE ON LAB HOST ONLY → VALIDATE
→ RELOAD (ONLY IF AUTHORIZED) → SECOND-SESSION TEST → ROLLBACK PRACTICE
→ FAILURES → INCIDENT → REFERENCE → DECISION TREE → FINAL CHALLENGE
```

# 3. Objectives

You will understand `/etc/ssh/ssh_config` versus `/etc/ssh/sshd_config`, drop-in directories, first-obtained client option behavior, server directives at a beginner level, `sshd -t` validation, `sshd -T` effective configuration, client `ssh -G`, safe backups, reload versus restart considerations, listener verification, second-session validation, and rollback discipline.

# 4. Lockout Prevention Rules

Before any server-side SSH change:

1. Keep the current working session open.
2. Confirm you have authorization.
3. Know whether out-of-band/console access exists.
4. Capture current effective configuration and listener state.
5. Back up the configuration you will edit.
6. Make one controlled change.
7. Run syntax validation before reload/restart.
8. Prefer reload when appropriate and supported; do not blindly restart.
9. Test a **new second session** before closing the original.
10. Roll back immediately if validation or access fails.

This lab must be performed on a disposable/authorized lab host for write operations. On production systems, keep the exercise read-only unless a real approved change exists.

# 5. Verify Tools and Service State

```bash
hostname
whoami
command -v ssh
command -v sshd || true
command -v systemctl
command -v ss
systemctl is-active ssh 2>/dev/null || true
ss -ltnp 'sport = :22' 2>/dev/null || ss -ltn 'sport = :22'
```

If `sshd` is unavailable, complete server-side sections using supplied evidence or an authorized lab VM. Do not install/start services blindly.

# 6. Client vs Server Configuration

```text
CLIENT                                 SERVER
------                                 ------
ssh                                    sshd
~/.ssh/config                          /etc/ssh/sshd_config
/etc/ssh/ssh_config                    /etc/ssh/sshd_config.d/*.conf
/etc/ssh/ssh_config.d/*.conf           server policy
```

Do not edit the server file to solve a client alias problem, or the client file to change the server listener.

# 7. Inspect Client Effective Configuration

```bash
ssh -G example.com | grep -E '^(hostname|user|port|identityfile|proxyjump|stricthostkeychecking) '
```

`ssh -G` evaluates configuration without making a connection. This is safer than guessing which stanza applies.

# 8. Understand Client Matching

A user config can contain:

```text
Host app-admin
    HostName app-admin-01.internal.example
    User opsuser
    Port 2222
```

This lab does not require adding it yet. Understand that `Host` selects a pattern/alias and later options influence the effective connection. Use `ssh -G app-admin` to see the result.

# 9. Inspect Server Configuration Read-Only

If authorized:

```bash
sudo grep -Ev '^\s*(#|$)' /etc/ssh/sshd_config
```

`sudo` is justified because server configuration may require elevated read access in some environments. Do not publish sensitive/internal policy unnecessarily.

Inspect drop-ins:

```bash
sudo find /etc/ssh/sshd_config.d -maxdepth 1 -type f -name '*.conf' -print 2>/dev/null
```

Ubuntu may use included configuration fragments. The effective result matters more than assuming one file owns every directive.

# 10. Validate Existing Server Syntax

Before editing anything:

```bash
sudo sshd -t
```

No output normally means syntax validation succeeded. A nonzero exit/error must be investigated before any reload.

Record the exit status immediately:

```bash
echo $?
```

# 11. Inspect Effective Server Configuration

```bash
sudo sshd -T | head -n 40
```

`-T` outputs effective configuration. To inspect selected values:

```bash
sudo sshd -T | grep -E '^(port|listenaddress|passwordauthentication|pubkeyauthentication|permitrootlogin) '
```

Do not assume defaults from memory; inspect the running version's effective configuration.

# 12. Record Pre-Change Evidence

```text
CURRENT SESSION SOURCE/DESTINATION:
CURRENT SSH PORT:
CURRENT LISTENER:
CURRENT EFFECTIVE Port:
CURRENT PasswordAuthentication:
CURRENT PubkeyAuthentication:
CURRENT PermitRootLogin:
OUT-OF-BAND ACCESS AVAILABLE?:
ROLLBACK OWNER:
```

Do not proceed with a write exercise if you cannot safely recover access.

# 13. Create a Backup Before an Authorized Lab Change

On a disposable lab host only:

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.veriqta-jun057.bak
sudo ls -l /etc/ssh/sshd_config /etc/ssh/sshd_config.veriqta-jun057.bak
```

`cp -a` preserves metadata. If your environment manages SSH through configuration management, the real source of truth may be elsewhere; do not create manual drift in production.

# 14. Prefer a Dedicated Drop-In for a Lab Change

Rather than rewriting the main file, an authorized Ubuntu lab can use a clearly named fragment when supported by the existing `Include` behavior.

First confirm include behavior:

```bash
sudo grep -n '^Include' /etc/ssh/sshd_config
```

Do not assume it exists.

For a disposable lab only, a harmless demonstrative directive can be placed in a dedicated file. For example, if policy permits and you understand the effect, set a login grace time rather than changing authentication methods:

```bash
printf 'LoginGraceTime 90\n' | sudo tee /etc/ssh/sshd_config.d/90-veriqta-jun057.conf
```

This changes server configuration. Do this **only** on the authorized lab host.

# 15. Validate Before Applying

```bash
sudo sshd -t
```

Then inspect effective value:

```bash
sudo sshd -T | grep '^logingracetime '
```

If validation fails, do not reload. Fix or remove the lab fragment while your original session remains open.

# 16. Apply With Minimum Disruption

If and only if the lab host is authorized and validation succeeded:

```bash
sudo systemctl reload ssh
```

Why reload? It asks the service to reread configuration without unnecessarily terminating the daemon process in the way a restart may. Exact service behavior depends on systemd/OpenSSH packaging; verify status.

```bash
systemctl is-active ssh
ss -ltn 'sport = :22'
```

# 17. Second-Session Verification

Do **not** close the original session.

Open a new terminal and connect using the expected account/port. Verify:

```bash
whoami
hostname
```

Only after a new session works do you have evidence that new connections remain possible.

# 18. Rollback the Lab Change

Remove only the lab fragment:

```bash
sudo rm /etc/ssh/sshd_config.d/90-veriqta-jun057.conf
sudo sshd -t
sudo systemctl reload ssh
```

Then verify the effective value and open another test session if appropriate.

This is deliberate rollback practice.

# 19. Controlled Syntax Failure Without Risking the Live Service

Do not put invalid syntax into the active configuration. Instead create a separate temporary candidate file:

```bash
cat > /tmp/veriqta-jun057-bad-sshd_config <<'EOF'
Port 22
ThisDirectiveDoesNotExist yes
EOF
```

Validate the candidate explicitly:

```bash
sudo sshd -t -f /tmp/veriqta-jun057-bad-sshd_config
```

Expected: validation error. Because you did not point the running service at this file, the failure is controlled.

Remove it after the exercise.

# 20. Understand Important Beginner Directives

- `Port`: listening port(s), subject to listener/firewall/policy implications.
- `ListenAddress`: addresses on which sshd listens.
- `PermitRootLogin`: controls root SSH login policy.
- `PasswordAuthentication`: controls password authentication where applicable.
- `PubkeyAuthentication`: controls public-key authentication.
- `AllowUsers` / `AllowGroups`: can restrict who may log in.
- `Include`: loads additional configuration files.

Do not change these casually. A syntactically valid setting can still lock out users.

# 21. Failure Lab — Syntax Error

Symptom: `sshd -t` reports an unknown directive. Correct action: do not reload; identify the offending file/line, correct it, rerun validation, and only then consider application.

# 22. Failure Lab — Valid Syntax, Wrong Policy

Supplied change:

```text
PasswordAuthentication no
PubkeyAuthentication yes
```

Syntax is valid. But if the intended admin account has no tested public key, this can cause lockout. Syntax validation is necessary, not sufficient.

# 23. Failure Lab — Wrong Port Expectation

Server effective config shows `port 2222`; client still targets 22. Do not edit firewall/authentication first. Align the expected port through approved configuration/inventory and verify listener/client behavior.

# 24. Failure Lab — Drop-In Overrides Assumption

An engineer edits `/etc/ssh/sshd_config`, but `sshd -T` still shows an unexpected effective value because an included fragment affects configuration. Troubleshoot the effective configuration and include order rather than repeatedly editing the main file.

# 25. Production Incident — Authentication Hardening Caused Lockout Risk

**Change request:** disable password authentication after key rollout.

**Impact:** one canary engineer reports public-key access works; several service accounts have not been verified.

### Evidence 1

```text
sshd -T | grep -E '^(passwordauthentication|pubkeyauthentication) '
passwordauthentication yes
pubkeyauthentication yes
```

### Proposed change

```text
PasswordAuthentication no
```

### Risk question

Is syntax validation enough? No. You must prove all required accounts have functioning alternative authentication before disabling passwords.

### Evidence 2 — inventory

```text
opsuser: key verified
backupsvc: key NOT verified
deploysvc: key verified
```

### Decision

Do not apply the global hardening change yet. The evidence shows a known access dependency for `backupsvc`.

### Authorized remediation plan

Provision and verify `backupsvc` key authentication through the approved process, test in a separate session/workflow, then schedule the hardening change with rollback and monitoring.

### Verification plan

1. `sshd -t` passes.
2. `sshd -T` shows intended effective policy.
3. service remains active/listening.
4. new sessions for required accounts succeed.
5. original sessions remain available until proof is complete.
6. authentication logs/monitoring show expected behavior.

### Prevention

Pre-change account inventory, canary testing, configuration-as-code, automated `sshd -t`, access-path tests, and documented rollback.



# Extended Follow-Along Engineering Practice — Safe SSH Configuration Change Discipline

This block deepens the same lab scope. Work through it in order; do not treat it as optional reading. For every command, record your own result and write one sentence stating what the evidence proves and one sentence stating what it does not prove.

## Practice 1 — Capture current SSH service state

A safe change begins with runtime evidence.

Run the following only on the authorized lab host/target:

```bash
systemctl is-active ssh 2>/dev/null || true
ss -ltn 'sport = :22'
```

Record what is actually active/listening before editing.

Record your evidence:

```text
SERVICE STATE:
LISTENER:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 2 — Capture effective client config

Client aliases/options can be part of an apparent server problem.

Run the following only on the authorized lab host/target:

```bash
ssh -G YOUR_HOST | grep -E '^(hostname|user|port|identityfile) '
```

Separate client configuration from server configuration.

Record your evidence:

```text
CLIENT HOSTNAME:
USER:
PORT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 3 — Capture effective server policy

Do not infer defaults from file comments.

Run the following only on the authorized lab host/target:

```bash
sudo sshd -T | grep -E '^(port|listenaddress|passwordauthentication|pubkeyauthentication|permitrootlogin) '
```

`sshd -T` gives effective values for the current context.

Record your evidence:

```text
PORT:
PASSWORD AUTH:
PUBKEY AUTH:
ROOT LOGIN:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 4 — Locate included fragments

Ubuntu configurations may be assembled from multiple files.

Run the following only on the authorized lab host/target:

```bash
sudo grep -n '^Include' /etc/ssh/sshd_config
sudo find /etc/ssh/sshd_config.d -maxdepth 1 -type f -name '*.conf' -print 2>/dev/null
```

Identify source ownership before editing.

Record your evidence:

```text
INCLUDE DIRECTIVE:
DROP-INS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 5 — Validate current config before change

If baseline validation already fails, do not attribute it to your proposed edit.

Run the following only on the authorized lab host/target:

```bash
sudo sshd -t
echo $?
```

Capture pre-change validation status.

Record your evidence:

```text
PRE-CHANGE VALID?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 6 — Back up with metadata

Backups support rollback but may not be the authoritative source in managed systems.

Run the following only on the authorized lab host/target:

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.veriqta-jun057.bak
sudo stat /etc/ssh/sshd_config.veriqta-jun057.bak
```

Verify backup existence/ownership.

Record your evidence:

```text
BACKUP PATH:
BACKUP OWNER/MODE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 7 — Validate an alternate file

Practice syntax failure without touching active config.

Run the following only on the authorized lab host/target:

```bash
sudo sshd -t -f /tmp/veriqta-jun057-bad-sshd_config
```

The error should identify invalid configuration; the live daemon remains pointed at its normal config.

Record your evidence:

```text
ERROR:
LIVE SERVICE IMPACT: NONE EXPECTED
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 8 — Compare intended and effective state

A text edit matters only if it produces the intended effective config.

Run the following only on the authorized lab host/target:

```bash
sudo sshd -t
sudo sshd -T | grep '^logingracetime '
```

Both parsing and effective-value checks are required.

Record your evidence:

```text
SYNTAX:
EFFECTIVE VALUE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 9 — Verify listener after authorized reload

Service-active alone is not enough.

Run the following only on the authorized lab host/target:

```bash
systemctl is-active ssh
ss -ltn 'sport = :22'
```

Confirm both service manager state and expected socket.

Record your evidence:

```text
SERVICE:
LISTENER:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 10 — Verify a second session

Existing sessions can survive a configuration that breaks new logins.

Run the following only on the authorized lab host/target:

```bash
ssh -o ConnectTimeout=5 USER@HOST 'whoami; hostname'
```

Run from a separate terminal while preserving the original session.

Record your evidence:

```text
NEW SESSION SUCCESS?:
REMOTE USER/HOST:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 11 — Compare backup and current config

Know exactly what changed.

Run the following only on the authorized lab host/target:

```bash
sudo diff -u /etc/ssh/sshd_config.veriqta-jun057.bak /etc/ssh/sshd_config || true
```

No diff may be expected if the lab change was in a drop-in. Inspect the correct source.

Record your evidence:

```text
DIFFERENCE:
SOURCE OF CHANGE:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 12 — Verify rollback end to end

Rollback is not complete when a file is copied; prove effective state and access.

Run the following only on the authorized lab host/target:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(port|passwordauthentication|pubkeyauthentication) '
systemctl is-active ssh
```

Then test a new session.

Record your evidence:

```text
VALIDATION:
EFFECTIVE STATE:
SERVICE:
NEW SESSION:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.




# Engineering Scenario Drill Bank

Use these as short incident rehearsals. For each scenario, do **not** jump directly to a fix. Write: symptom, expected state, first evidence to collect, two plausible hypotheses, one discriminating test for each hypothesis, safe remediation owner, verification of the original operation, and one prevention control.

## Drill 1 — Syntax Error In Candidate Config

**Scenario:** The production symptom is **syntax error in candidate config**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: syntax error in candidate config
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

## Drill 2 — Valid Syntax Disables Required Auth Path

**Scenario:** The production symptom is **valid syntax disables required auth path**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: valid syntax disables required auth path
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

## Drill 3 — Wrong Port Directive

**Scenario:** The production symptom is **wrong Port directive**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: wrong Port directive
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

## Drill 4 — Wrong Listenaddress

**Scenario:** The production symptom is **wrong ListenAddress**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: wrong ListenAddress
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

## Drill 5 — Allowusers Excludes Admin

**Scenario:** The production symptom is **AllowUsers excludes admin**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: AllowUsers excludes admin
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

## Drill 6 — Drop-In Changes Effective Value

**Scenario:** The production symptom is **drop-in changes effective value**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: drop-in changes effective value
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

## Drill 7 — Manual Edit Overwritten By Automation

**Scenario:** The production symptom is **manual edit overwritten by automation**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: manual edit overwritten by automation
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

## Drill 8 — Reload Succeeds But New Login Fails

**Scenario:** The production symptom is **reload succeeds but new login fails**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: reload succeeds but new login fails
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

## Drill 9 — Listener Absent After Change

**Scenario:** The production symptom is **listener absent after change**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: listener absent after change
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

## Drill 10 — Rollback File Restored But Effective State Not Verified

**Scenario:** The production symptom is **rollback file restored but effective state not verified**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: rollback file restored but effective state not verified
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


# 26. Command-and-Directive Reference

| Command | Purpose |
|---|---|
| `ssh -G HOST` | Print effective client configuration |
| `sshd -t` | Validate server configuration syntax |
| `sshd -T` | Print effective server configuration |
| `sshd -t -f FILE` | Validate an alternate candidate file |
| `systemctl is-active ssh` | Check service active state |
| `systemctl reload ssh` | Reload configuration when authorized/validated |
| `ss -ltn 'sport = :22'` | Verify expected listener without re-teaching socket inspection |
| `cp -a SOURCE BACKUP` | Preserve metadata in a controlled backup |
| `diff -u A B` | Compare configuration versions |

| Directive | Meaning / risk |
|---|---|
| `Port` | SSH listener port; changing it affects clients/network policy |
| `ListenAddress` | Bind address; wrong value can remove reachability |
| `PermitRootLogin` | Root login policy |
| `PasswordAuthentication` | Password auth policy; disabling before alternatives are verified can lock users out |
| `PubkeyAuthentication` | Public-key auth policy |
| `AllowUsers` | Restricts permitted users; errors can lock out admins |
| `AllowGroups` | Restricts permitted groups |
| `Include` | Loads additional configuration fragments |

# 27. SSH Configuration Troubleshooting Decision Tree

```text
SSH CONFIG CHANGE PLANNED
        |
        v
WORKING SESSION + ROLLBACK PATH?
 |                     |
NO                    YES
 |                     v
STOP              CAPTURE BASELINE
                       |
                       v
                  BACKUP/SOURCE OF TRUTH
                       |
                       v
                    EDIT ONE CHANGE
                       |
                       v
                    sshd -t PASSES?
                    |           |
                   NO          YES
                    |           v
                 DO NOT      sshd -T EXPECTED?
                 RELOAD       |          |
                             NO         YES
                              |          v
                           FIX PLAN   AUTHORIZED RELOAD
                                         |
                                         v
                                  SECOND SESSION WORKS?
                                     |           |
                                    NO          YES
                                     |           v
                                  ROLLBACK    VERIFY + MONITOR
```

# 28. Independent Final Challenge — Safe SSH Configuration Change

On a disposable authorized Ubuntu lab host, choose one low-risk SSH configuration change approved for the exercise. You must capture pre-change effective config and listener state, identify the configuration owner/source, back it up, make one change, validate with `sshd -t`, confirm with `sshd -T`, reload only after validation, test a new session while keeping the old session open, and perform a rollback.

Report:

```text
LAB: JUN-057
HOST:
CHANGE TICKET/EXERCISE AUTHORIZATION:
CURRENT ACCESS PATH:
OUT-OF-BAND/ROLLBACK PATH:
PRE-CHANGE LISTENER:
PRE-CHANGE EFFECTIVE CONFIG:
SOURCE FILE/DROP-IN:
BACKUP:
INTENDED CHANGE:
RISK/BLAST RADIUS:
SYNTAX VALIDATION:
EFFECTIVE-CONFIG VALIDATION:
APPLY METHOD:
SERVICE/LISTENER POST-CHECK:
SECOND-SESSION TEST:
ORIGINAL OPERATION TEST:
ROLLBACK STEPS:
ROLLBACK TEST:
PREVENTION/AUTOMATION:
LIMITATIONS:
```

### Acceptance Criteria

- [ ] Existing access preserved throughout.
- [ ] Recovery/rollback path established before change.
- [ ] Current effective configuration captured.
- [ ] Source of truth identified.
- [ ] Backup created where appropriate.
- [ ] One controlled change made.
- [ ] `sshd -t` passed before reload.
- [ ] `sshd -T` confirmed intended effective state.
- [ ] New second session verified before old session closed.
- [ ] Rollback was documented and, in the lab, safely practiced.
- [ ] No security control was disabled merely to make access work.

# 29. Cleanup

Remove only temporary lab artifacts after verifying paths:

```bash
rm -f /tmp/veriqta-jun057-bad-sshd_config
```

If the dedicated lab drop-in still exists, remove it only on the authorized lab host, validate, reload, and verify the restored effective state. Retain backups according to the exercise/change policy rather than deleting production evidence casually.

---

# Knowledge Check

### 1. What is the main client config file for a user?

**Answer:** `~/.ssh/config`.

### 2. What is the main server config file?

**Answer:** `/etc/ssh/sshd_config`.

### 3. What does `ssh -G` do?

**Answer:** Prints effective client configuration without connecting.

### 4. What does `sshd -t` do?

**Answer:** Validates server configuration syntax.

### 5. What does `sshd -T` do?

**Answer:** Prints effective server configuration.

### 6. Why keep the old session open?

**Answer:** It preserves access if new connections fail.

### 7. Why is syntax validation insufficient?

**Answer:** A valid policy can still deny required users or bind the wrong port/address.

### 8. Why inspect drop-ins?

**Answer:** Included fragments may affect effective configuration.

### 9. What is the safest time to reload?

**Answer:** After authorization, backup/source-of-truth confirmation, syntax validation, and risk review.

### 10. Why prefer a second-session test?

**Answer:** It proves new connections work without sacrificing the existing recovery session.

### 11. What does Port change affect?

**Answer:** Client expectations and potentially network/firewall policy.

### 12. What risk does ListenAddress carry?

**Answer:** Binding the wrong address can make SSH unreachable.

### 13. What risk does AllowUsers carry?

**Answer:** Required administrators can be excluded.

### 14. What risk does disabling PasswordAuthentication carry?

**Answer:** Accounts without verified alternative auth can be locked out.

### 15. Why use `sshd -T` after editing?

**Answer:** To confirm the effective result, not just file text.

### 16. Why identify configuration ownership?

**Answer:** Manual edits may be overwritten or create drift if automation manages the source.

### 17. Why practice rollback?

**Answer:** A rollback plan that has never been tested may fail during an outage.

### 18. Reload or restart?

**Answer:** Use the least disruptive approved method; validate first and understand platform behavior.

### 19. Should you disable host-key checking to recover access?

**Answer:** No; that weakens identity verification and does not solve server config safely.

### 20. What is the safe change sequence?

**Answer:** Protect access → baseline → backup/source → one change → validate → effective check → apply → second session → verify → monitor/rollback.

---

# Interview Preparation

### 1. How do you safely change sshd configuration remotely?

**Worked answer:** Keep a working session, ensure rollback/console access, capture baseline/effective config, back up or edit managed source, make one change, run `sshd -t`, confirm `sshd -T`, reload if authorized, test a second session, then verify and monitor.

### 2. What is the difference between ssh_config and sshd_config?

**Worked answer:** ssh_config controls clients; sshd_config controls the server daemon.

### 3. Why is `sshd -t` critical?

**Worked answer:** It catches syntax/configuration parsing errors before applying them to the service.

### 4. Why is `sshd -T` also important?

**Worked answer:** It shows effective values after defaults/includes, revealing what sshd will actually use.

### 5. How can a valid change still cause lockout?

**Worked answer:** It may disable the only working auth method, restrict users, change port, or bind an unreachable address.

### 6. How do you handle configuration management?

**Worked answer:** Change the authoritative managed source rather than creating untracked manual drift, then verify rendered/effective state.

### 7. What is your rollback trigger?

**Worked answer:** Failed validation, unexpected effective config, service/listener failure, or failed second-session access.

### 8. Why not close the original session after reload?

**Worked answer:** A reload can leave existing sessions alive even when new sessions are broken; the original session is your recovery path.

### 9. What should an SSH change report include?

**Worked answer:** Baseline, source, backup, intended change, risk, validation, apply method, second-session test, rollback, verification, prevention.

### 10. What is the key operational principle?

**Worked answer:** SSH configuration changes are access changes; prove safety before sacrificing the access path.

---

# Portfolio Evidence

Create a sanitized artifact named `jun-057-engineering-report.md`. Include the problem statement, baseline, commands selected, important observations, reasoning, failure evidence, recovery or remediation design, verification, and prevention. Remove internal hostnames, addresses, usernames, keys, tokens, and other sensitive infrastructure details before publishing.

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

## Next Lab — JUN-058 — Bash Fundamentals

Continue to the next lab only after you can explain the evidence you collected here without relying on memorized command sequences.
