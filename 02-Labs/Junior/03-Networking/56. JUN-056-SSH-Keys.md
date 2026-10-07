# JUN-056 — SSH Keys

**VERIQTA | LINUX WORLD — HANDS-ON ENGINEERING LAB**

**Suggested repository path:** `Junior/03-Networking/JUN-056-SSH-Keys/README.md`

| Field | Lab Standard |
|---|---|
| Lab ID | JUN-056 |
| Track | Junior Engineer |
| Topic | SSH key pairs, public-key authentication, permissions, fingerprints, agents, and safe key lifecycle fundamentals |
| Difficulty | 4/5 |
| Estimated Duration | 270–300 minutes |
| Operating System | Ubuntu Server 24.04 LTS |
| Required Privileges | Standard user; `sudo` only where explicitly justified |
| Primary Tools | `ssh-keygen`, `ssh`, `ssh-add`, `stat`, `chmod` |
| Lab Style | Guided follow-along engineering investigation |
| Final Deliverable | SSH Key Authentication Evidence Report |
| Next Lab | JUN-057 — SSH Configuration Basics |

> **Evidence rule:** Outputs shown in this lab are labeled illustrative, expected, or supplied incident evidence. Your own report must contain what you actually observed. If something is absent, record **Not observed** rather than inventing evidence.

---
# 1. Production Mission

Your team is moving an administrator from password-based access to public-key authentication on a controlled lab host. You must create a key pair safely, understand exactly which half is secret, verify fingerprints and permissions, install only the public key on an authorized account, test in a second session without sacrificing existing access, diagnose common key failures, and document lifecycle responsibilities.

This lab does not redesign server-wide SSH policy. That belongs to JUN-057.

# 2. Workflow

```text
PRE-CHECK → CREATE CONTROLLED WORKSPACE → GENERATE KEY → IDENTIFY PRIVATE/PUBLIC
→ INSPECT PERMISSIONS → FINGERPRINT → OPTIONAL AGENT → INSTALL PUBLIC KEY SAFELY
→ TEST SECOND SESSION → CONTROLLED FAILURES → INCIDENT → REFERENCE
→ DECISION TREE → FINAL CHALLENGE → CLEANUP
```

# 3. Objectives

You will understand asymmetric key pairs, Ed25519 keys, passphrases, public-key fingerprints, `authorized_keys`, private-key permissions, `ssh-copy-id` where appropriate, explicit `-i` identity selection, `IdentitiesOnly`, ssh-agent basics, revocation/removal concepts, and evidence-based diagnosis of public-key failures.

# 4. Non-Negotiable Key Safety

- A **private key is secret**. Never send it to a server, teammate, Git repository, ticket, chat, or portfolio.
- A public key is designed to be shared with authorized systems, but still reveals identity/comment metadata.
- Never overwrite an existing key pair casually.
- Use a dedicated lab key with an explicit filename.
- Protect your working SSH session while testing.
- Never remove existing authorized access until the replacement path is proven in a separate session.

# 5. Verify Tools

```bash
hostname
whoami
command -v ssh
command -v ssh-keygen
command -v ssh-add
command -v stat
```

Create a controlled workspace:

```bash
mkdir -p /tmp/veriqta-jun-056
chmod 700 /tmp/veriqta-jun-056
cd /tmp/veriqta-jun-056
pwd
```

# 6. Generate a Dedicated Ed25519 Lab Key

Run:

```bash
ssh-keygen -t ed25519 -a 64 -f /tmp/veriqta-jun-056/veriqta_jun056_ed25519 -C 'veriqta-jun-056-lab'
```

When prompted, choose a passphrase appropriate for your lab/security requirements. Do not paste the passphrase into notes.

Flags:

- `-t ed25519` selects Ed25519.
- `-a 64` increases KDF rounds used when protecting the private key with a passphrase.
- `-f` chooses an explicit file path.
- `-C` adds a comment to the public key.

# 7. Identify the Two Files

```bash
ls -l /tmp/veriqta-jun-056
```

You should have:

```text
veriqta_jun056_ed25519       ← PRIVATE KEY
veriqta_jun056_ed25519.pub   ← PUBLIC KEY
```

Do not `cat` the private key into shared logs or screenshots.

It is acceptable to inspect the public key in this controlled lab:

```bash
cat /tmp/veriqta-jun-056/veriqta_jun056_ed25519.pub
```

# 8. Inspect Permissions

```bash
stat -c '%A %a %U:%G %n' /tmp/veriqta-jun-056/veriqta_jun056_ed25519*
```

OpenSSH normally creates the private key with restrictive permissions. The public key can be readable. Record actual values.

# 9. Calculate the Public-Key Fingerprint

```bash
ssh-keygen -lf /tmp/veriqta-jun-056/veriqta_jun056_ed25519.pub
```

`-l` shows fingerprint information; `-f` identifies the key file.

Record:

```text
KEY TYPE:
BITS/TYPE FIELD:
FINGERPRINT:
COMMENT:
```

The fingerprint is safer to compare than visually comparing long key material.

# 10. Verify Private/Public Relationship

Derive the public key from the private key without replacing files:

```bash
ssh-keygen -y -f /tmp/veriqta-jun-056/veriqta_jun056_ed25519 > /tmp/veriqta-jun-056/derived.pub
```

`-y` reads a private OpenSSH key and prints its public key. Compare key material while accounting for comments:

```bash
cut -d' ' -f1-2 /tmp/veriqta-jun-056/veriqta_jun056_ed25519.pub > /tmp/veriqta-jun-056/original.material
cut -d' ' -f1-2 /tmp/veriqta-jun-056/derived.pub > /tmp/veriqta-jun-056/derived.material
diff -u /tmp/veriqta-jun-056/original.material /tmp/veriqta-jun-056/derived.material
```

No diff output means the key type/base64 public material matches.

# 11. Understand authorized_keys

On a server account, public keys commonly appear in:

```text
~/.ssh/authorized_keys
```

The server stores authorized **public** keys. It should not receive your private key.

Typical secure ownership/permissions are restrictive on the user's `.ssh` directory and `authorized_keys`, but exact server policy may vary. Never loosen permissions blindly.

# 12. Install the Public Key Only on an Authorized Lab Target

If you have an approved test host and an existing authenticated path, `ssh-copy-id` may be available:

```bash
command -v ssh-copy-id
```

If available and authorized:

```bash
ssh-copy-id -i /tmp/veriqta-jun-056/veriqta_jun056_ed25519.pub USER@HOST
```

This sends the public key, not the private key. If the tool is unavailable, follow your organization's approved provisioning method rather than inventing a production workaround.

# 13. Test With an Explicit Identity

Keep your existing session open. In a second terminal:

```bash
ssh -i /tmp/veriqta-jun-056/veriqta_jun056_ed25519 USER@HOST
```

`-i` selects the identity file. If the agent or default key discovery creates ambiguity, use:

```bash
ssh -o IdentitiesOnly=yes -i /tmp/veriqta-jun-056/veriqta_jun056_ed25519 USER@HOST
```

This asks SSH to use the explicitly configured identities rather than offering many agent/default keys.

# 14. Verify the Session Before Removing Anything

Inside the new session, verify identity and host:

```bash
whoami
hostname
```

Only after the new access path is independently verified should any migration plan consider retiring older credentials—and that retirement must be authorized. This lab does not remove production credentials.

# 15. ssh-agent Fundamentals

Check whether an agent is available:

```bash
ssh-add -l
```

Possible result: no identities, a list of identities, or no agent connection.

If your lab shell already has an agent and policy permits, add the dedicated key:

```bash
ssh-add /tmp/veriqta-jun-056/veriqta_jun056_ed25519
```

Then:

```bash
ssh-add -l
```

Do not treat an agent as a place to store keys permanently. Understand your environment's session/lifecycle behavior.

# 16. Remove the Lab Key From the Agent

If you added it:

```bash
ssh-add -d /tmp/veriqta-jun-056/veriqta_jun056_ed25519
```

Verify with `ssh-add -l`.

# 17. Controlled Failure — Private Key Permissions

Make a copy of the lab private key, not your real key:

```bash
cp /tmp/veriqta-jun-056/veriqta_jun056_ed25519 /tmp/veriqta-jun-056/insecure-copy
chmod 0644 /tmp/veriqta-jun-056/insecure-copy
stat -c '%A %a %n' /tmp/veriqta-jun-056/insecure-copy
```

If you use this copy with SSH against an authorized target, OpenSSH may refuse it because permissions are too open. Do not depend on exact wording across versions.

Restore safe permissions on the copy:

```bash
chmod 0600 /tmp/veriqta-jun-056/insecure-copy
```

The lesson: permission checks protect private-key secrecy.

# 18. Controlled Failure — Wrong Key

Generate a second disposable key:

```bash
ssh-keygen -t ed25519 -f /tmp/veriqta-jun-056/wrong_key -N '' -C 'wrong-key-lab'
```

Against a target that authorizes only the first lab public key, test explicitly:

```bash
ssh -o IdentitiesOnly=yes -i /tmp/veriqta-jun-056/wrong_key USER@HOST
```

Expected on such a target: authentication failure. Transport and host verification may still succeed.

# 19. Failure Lab — Too Many Identities

Supplied evidence:

```text
Received disconnect ... Too many authentication failures
```

A client/agent may offer multiple keys before the correct one. Use verbose output to confirm offered identities, then explicitly select the intended key and `IdentitiesOnly=yes` when appropriate. Do not delete unrelated keys as a first response.

# 20. Failure Lab — Public Key Not Authorized

Supplied evidence:

```text
Offering public key: ...
Authentications that can continue: publickey
Permission denied (publickey).
```

Possible hypotheses: wrong user; public key absent from correct account; permissions/ownership prevent server use; server policy disallows the algorithm/method; wrong identity offered. Gather evidence rather than regenerating keys immediately.

# 21. Failure Lab — Wrong Username

The correct public key can be authorized for `deploy`, while the engineer attempts `ubuntu@host`. Authentication fails because authorization is account-specific. Verify intended user explicitly.

# 22. Production Incident — Key Rotation Broke Automation

**Impact:** deployment automation can no longer SSH to `release-host-01`.

### Evidence 1

```text
Permission denied (publickey).
```

### Evidence 2 — verbose client summary

```text
Offering public key: /home/runner/.ssh/release_old ED25519 SHA256:OLD...
Server accepts key: no
```

### Evidence 3 — approved rotation record

```text
New key fingerprint: SHA256:NEW...
Old key removal approved after 18:00 UTC
Automation secret should have been updated before removal.
```

### Evidence 4 — server authorization inventory

```text
authorized key fingerprint present: SHA256:NEW...
old fingerprint absent
```

Root cause: the server rotation completed, but the automation client still used the retired private key.

### Remediation

Update the automation's secret/credential reference through the approved secret-management process. Never copy the new private key into a Git repository or ticket.

### Verification

Run the exact deployment connectivity check with the intended automation identity, confirm the correct fingerprint is used, and verify the deployment operation—not merely interactive SSH.

### Prevention

Overlap key rotations, inventory fingerprints, validate clients before revocation, use secret-management workflows, and monitor authentication failures.

# 23. Key Lifecycle Mental Model

```text
GENERATE
  ↓
PROTECT PRIVATE KEY
  ↓
DISTRIBUTE PUBLIC KEY
  ↓
VERIFY FINGERPRINT
  ↓
AUTHORIZE ACCOUNT
  ↓
TEST NEW ACCESS
  ↓
MONITOR/INVENTORY
  ↓
ROTATE
  ↓
REVOKE OLD PUBLIC AUTHORIZATION
  ↓
DESTROY RETIRED PRIVATE MATERIAL SECURELY PER POLICY
```



# Extended Follow-Along Engineering Practice — SSH Key Evidence and Lifecycle

This block deepens the same lab scope. Work through it in order; do not treat it as optional reading. For every command, record your own result and write one sentence stating what the evidence proves and one sentence stating what it does not prove.

## Practice 1 — Inventory without exposing keys

Know what files exist before generating anything.

Run the following only on the authorized lab host/target:

```bash
find ~/.ssh -maxdepth 1 -type f -printf '%f\n' 2>/dev/null | sort
```

File names can still be sensitive; do not publish them blindly. Never print private-key contents.

Record your evidence:

```text
FILES OBSERVED:
DEDICATED LAB KEY COLLISION?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 2 — Fingerprint the lab public key

Fingerprints are safer inventory identifiers than copying key blobs.

Run the following only on the authorized lab host/target:

```bash
ssh-keygen -lf /tmp/veriqta-jun-056/veriqta_jun056_ed25519.pub
```

Record type/comment/fingerprint.

Record your evidence:

```text
FINGERPRINT:
TYPE:
COMMENT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 3 — Inspect private-key permissions

Permission evidence belongs in the report.

Run the following only on the authorized lab host/target:

```bash
stat -c '%A %a %U:%G %n' /tmp/veriqta-jun-056/veriqta_jun056_ed25519
```

Do not loosen permissions to troubleshoot.

Record your evidence:

```text
MODE:
OWNER:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 4 — Compare public material

Prove the public file corresponds to the private key without sharing the private key.

Run the following only on the authorized lab host/target:

```bash
ssh-keygen -y -f /tmp/veriqta-jun-056/veriqta_jun056_ed25519 | cut -d' ' -f1-2
cut -d' ' -f1-2 /tmp/veriqta-jun-056/veriqta_jun056_ed25519.pub
```

Matching type/base64 material supports that the pair corresponds.

Record your evidence:

```text
MATCH?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 5 — Inspect agent state before modifying it

Do not assume the agent contains the key.

Run the following only on the authorized lab host/target:

```bash
ssh-add -l
```

Record no-agent/no-identities/listed-identities accurately.

Record your evidence:

```text
AGENT RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 6 — Add only the lab identity

If an agent is available and policy permits, add the dedicated key.

Run the following only on the authorized lab host/target:

```bash
ssh-add /tmp/veriqta-jun-056/veriqta_jun056_ed25519
ssh-add -l
```

Confirm by fingerprint rather than filename alone.

Record your evidence:

```text
LAB FINGERPRINT PRESENT?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 7 — Use explicit identity selection

Eliminate ambiguity from default keys.

Run the following only on the authorized lab host/target:

```bash
ssh -o IdentitiesOnly=yes -i /tmp/veriqta-jun-056/veriqta_jun056_ed25519 USER@HOST
```

This test should be run only after the public key is authorized.

Record your evidence:

```text
TARGET USER:
IDENTITY FINGERPRINT:
AUTH RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 8 — Inspect public authorization safely

On an authorized lab account, inventory public-key fingerprints rather than copying private material.

Run the following only on the authorized lab host/target:

```bash
ssh USER@HOST 'test -f ~/.ssh/authorized_keys && ssh-keygen -lf ~/.ssh/authorized_keys || true'
```

Output may reveal public key comments/fingerprints; sanitize before publishing.

Record your evidence:

```text
EXPECTED FINGERPRINT PRESENT?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 9 — Demonstrate wrong-user failure

Authorization is per account.

Run the following only on the authorized lab host/target:

```bash
ssh -o IdentitiesOnly=yes -i /tmp/veriqta-jun-056/veriqta_jun056_ed25519 WRONG_USER@HOST
```

Use only a known lab account/target and avoid repeated attempts.

Record your evidence:

```text
RESULT:
WHY ACCOUNT MATTERS:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 10 — Demonstrate wrong-key evidence

A wrong key should not be “fixed” by weakening server policy.

Run the following only on the authorized lab host/target:

```bash
ssh -v -o IdentitiesOnly=yes -i /tmp/veriqta-jun-056/wrong_key USER@HOST
```

Identify the offered fingerprint and authentication result.

Record your evidence:

```text
OFFERED KEY:
AUTH RESULT:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 11 — Remove the agent identity cleanly

Undo only the lab agent change.

Run the following only on the authorized lab host/target:

```bash
ssh-add -d /tmp/veriqta-jun-056/veriqta_jun056_ed25519
ssh-add -l
```

Confirm the intended key was removed without clearing unrelated identities.

Record your evidence:

```text
REMOVED?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.

## Practice 12 — Write a rotation proof checklist

A key rotation is complete only when the new path is proven before old revocation.

Run the following only on the authorized lab host/target:

```bash
ssh -o IdentitiesOnly=yes -i NEW_KEY USER@HOST 'whoami; hostname'
```

Replace NEW_KEY only in a real authorized rotation. Record proof before revocation.

Record your evidence:

```text
NEW KEY VERIFIED?:
OLD KEY REVOCATION APPROVED?:
WHAT THIS PROVES:
WHAT THIS DOES NOT PROVE:
NEXT QUESTION:
```

### Checkpoint

Before moving on, explain why the command above answers a specific question rather than serving as command roulette. If the result differs from the illustrative expectation, keep the real result and investigate the difference.




# Engineering Scenario Drill Bank

Use these as short incident rehearsals. For each scenario, do **not** jump directly to a fix. Write: symptom, expected state, first evidence to collect, two plausible hypotheses, one discriminating test for each hypothesis, safe remediation owner, verification of the original operation, and one prevention control.

## Drill 1 — Wrong Private-Key Permissions

**Scenario:** The production symptom is **wrong private-key permissions**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: wrong private-key permissions
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

## Drill 2 — Wrong Public Key Installed

**Scenario:** The production symptom is **wrong public key installed**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: wrong public key installed
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

## Drill 3 — Correct Key On Wrong Account

**Scenario:** The production symptom is **correct key on wrong account**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: correct key on wrong account
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

## Drill 4 — Agent Offers Too Many Keys

**Scenario:** The production symptom is **agent offers too many keys**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: agent offers too many keys
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

## Drill 5 — Explicit Identity Not Selected

**Scenario:** The production symptom is **explicit identity not selected**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: explicit identity not selected
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

## Drill 6 — Public Key Removed During Rotation

**Scenario:** The production symptom is **public key removed during rotation**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: public key removed during rotation
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

## Drill 7 — Automation Still Uses Retired Key

**Scenario:** The production symptom is **automation still uses retired key**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: automation still uses retired key
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

## Drill 8 — Private Key Path Missing

**Scenario:** The production symptom is **private key path missing**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: private key path missing
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

## Drill 9 — Fingerprint Inventory Mismatch

**Scenario:** The production symptom is **fingerprint inventory mismatch**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: fingerprint inventory mismatch
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

## Drill 10 — New Key Works Interactively But Automation Still Fails

**Scenario:** The production symptom is **new key works interactively but automation still fails**. Treat that sentence as the only fact initially supplied.

Complete this worksheet before reading or inventing a remediation:

```text
SYMPTOM: new key works interactively but automation still fails
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
| `ssh-keygen -t ed25519 ...` | Generate Ed25519 key pair |
| `ssh-keygen -lf KEY.pub` | Display public-key fingerprint |
| `ssh-keygen -y -f PRIVATE` | Derive public material from private key |
| `ssh -i PRIVATE USER@HOST` | Use explicit identity |
| `ssh -o IdentitiesOnly=yes ...` | Restrict offered identities |
| `ssh-add -l` | List agent identities |
| `ssh-add KEY` | Add key to agent |
| `ssh-add -d KEY` | Remove key from agent |
| `stat -c ...` | Inspect permissions/ownership metadata |

| Flag | Meaning |
|---|---|
| `-t` | Key type |
| `-a` | KDF rounds for passphrase-protected private key format |
| `-f` | File path |
| `-C` | Public-key comment |
| `-N` | New passphrase argument; avoid exposing real passphrases in shell history |
| `-l` | Fingerprint/list depending on command context |
| `-y` | Print public key from private key |
| `-i` | SSH identity file |

# 25. SSH Key Troubleshooting Decision Tree

```text
PUBLIC-KEY AUTH FAILS
        |
        v
DID TCP/HOST VERIFICATION SUCCEED?
 |                    |
NO                   YES
 |                    v
JUN-055 layers    CORRECT USER?
                     |      |
                    NO     YES
                     |      v
                 fix user  CORRECT KEY OFFERED?
                              |       |
                             NO      YES
                              |       v
                       select identity PUBLIC KEY AUTHORIZED?
                                         |       |
                                        NO      YES
                                         |       v
                                   provision    permissions/policy/logs
                                   public key
```

# 26. Independent Final Challenge — Key Authentication Evidence

Using only an authorized lab account, create a dedicated Ed25519 key, record its fingerprint and permissions, provision only its public key through an approved method, verify access in a second session with explicit identity selection, demonstrate one controlled failure, recover it, and remove the lab authorization if your environment permits safe cleanup.

Report:

```text
LAB: JUN-056
CLIENT HOST:
TARGET ACCOUNT:
KEY TYPE:
PUBLIC FINGERPRINT:
PRIVATE KEY LOCATION:
PRIVATE KEY PERMISSIONS:
PUBLIC KEY LOCATION:
PROVISIONING METHOD:
SECOND-SESSION TEST:
CONTROLLED FAILURE:
EVIDENCE:
ROOT CAUSE:
RECOVERY:
VERIFICATION:
AGENT USE:
ROTATION/REVOCATION PLAN:
SECRETS EXPOSED?: NO
LIMITATIONS:
```

### Acceptance Criteria

- [ ] Dedicated key used; no existing key overwritten.
- [ ] Private key never copied into the report.
- [ ] Fingerprint recorded.
- [ ] Private permissions inspected.
- [ ] Only public material provisioned to server.
- [ ] New access verified in a second session.
- [ ] Existing access preserved until verification.
- [ ] Controlled failure investigated with evidence.
- [ ] Agent state cleaned up if modified.
- [ ] Lab authorization removed only when safe/authorized.

# 27. Cleanup

If you added the lab key to an agent, remove it. If you installed the lab public key on an authorized disposable account, remove only that exact public-key line using the approved method after confirming another access path exists. Then remove the local temporary workspace only after confirming it contains only lab material:

```bash
ls -la /tmp/veriqta-jun-056
rm -rf /tmp/veriqta-jun-056
```

Never use this cleanup sequence on your real `~/.ssh` directory.

---

# Knowledge Check

### 1. Which key must remain secret?

**Answer:** The private key.

### 2. Which key is placed in authorized_keys?

**Answer:** The public key.

### 3. What is a fingerprint?

**Answer:** A compact cryptographic identifier derived from a key, useful for comparison/inventory.

### 4. Why use a dedicated filename?

**Answer:** To avoid overwriting or confusing existing credentials.

### 5. What does `-t ed25519` do?

**Answer:** Selects the Ed25519 key type.

### 6. What does `-C` do?

**Answer:** Adds a comment to the public key.

### 7. What does `ssh -i` do?

**Answer:** Selects an identity file.

### 8. What does IdentitiesOnly help with?

**Answer:** It limits which identities SSH offers, reducing ambiguity/too-many-key problems.

### 9. Why protect private-key permissions?

**Answer:** Other users should not be able to read secret key material; OpenSSH may reject overly open files.

### 10. What is authorized_keys?

**Answer:** A server-account file listing public keys allowed for authentication, subject to server policy.

### 11. Is a public key a password?

**Answer:** No; it is public cryptographic material paired with the private key.

### 12. Why use a passphrase?

**Answer:** It adds protection if the private-key file is stolen, subject to automation/use-case design.

### 13. What does ssh-agent do?

**Answer:** Holds decrypted key identities for a session/process context so repeated passphrase entry may be avoided.

### 14. Should agents be treated as permanent storage?

**Answer:** No.

### 15. Why test a rotated key before removing the old one?

**Answer:** To avoid lockout/outage.

### 16. What can cause publickey permission denied?

**Answer:** Wrong user/key, missing authorization, permissions/ownership, server policy, or client offering behavior.

### 17. Why not regenerate keys immediately after failure?

**Answer:** It can hide the actual authorization/configuration problem and complicate inventory.

### 18. What is safe to publish?

**Answer:** Only sanitized evidence; never private key material or sensitive infrastructure data.

### 19. How should automation key rotation be done?

**Answer:** Update/prove new client credential before revoking old authorization, with controlled overlap and inventory.

### 20. What is the key lifecycle?

**Answer:** Generate, protect, distribute public key, authorize, test, inventory, rotate, revoke, retire securely.

---

# Interview Preparation

### 1. Explain SSH public-key authentication.

**Worked answer:** The client proves possession of a private key corresponding to a public key authorized for the target account; the private key remains on the client.

### 2. How do you troubleshoot Permission denied publickey?

**Worked answer:** Confirm connection/host verification, exact username, offered identity via verbose logs, server authorization, permissions/ownership, and policy before changing keys.

### 3. Why use IdentitiesOnly?

**Worked answer:** It prevents unrelated agent/default identities from being offered when an explicit identity should be used.

### 4. How do you rotate keys safely?

**Worked answer:** Provision and verify the new public key/client secret first, then revoke old authorization after successful testing and rollback planning.

### 5. What is the risk of copying private keys into CI configuration manually?

**Worked answer:** Secret leakage, poor auditability, uncontrolled duplication, and difficult revocation; use approved secret management.

### 6. How do you verify a key pair matches?

**Worked answer:** Derive/compare public material or compare fingerprints without exposing the private key.

### 7. What permissions should a private key have?

**Worked answer:** Restrictive owner-only access is typical; exact policy/environment matters, and OpenSSH rejects overly open keys.

### 8. What is the difference between host keys and user keys?

**Worked answer:** Host keys authenticate the server; user keys authenticate a user/client identity.

### 9. What do you document for key inventory?

**Worked answer:** Owner/purpose, key type, fingerprint, authorized accounts/systems, creation/rotation dates, and revocation status—not private material.

### 10. What is the most important key rule?

**Worked answer:** The private key never leaves its approved secret boundary.

---

# Portfolio Evidence

Create a sanitized artifact named `jun-056-engineering-report.md`. Include the problem statement, baseline, commands selected, important observations, reasoning, failure evidence, recovery or remediation design, verification, and prevention. Remove internal hostnames, addresses, usernames, keys, tokens, and other sensitive infrastructure details before publishing.

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

## Next Lab — JUN-057 — SSH Configuration Basics

Continue to the next lab only after you can explain the evidence you collected here without relying on memorized command sequences.
