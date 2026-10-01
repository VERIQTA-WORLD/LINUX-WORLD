# JUN-041 — `/etc/fstab`

- **Level:** Junior  
- **Domain:** Linux Storage Administration  
- **Environment:** Ubuntu Server 24.04 LTS  
- **Prerequisite:** JUN-040 — Mounting and Unmounting Filesystems  
- **Primary Configuration File:** `/etc/fstab`

---

## 1. Mission

In JUN-040, you learned that a filesystem can be attached to the Linux directory tree with `mount` and detached with `umount`.

That raises an important operational question:

**How does Linux know which filesystems should be mounted automatically when the system starts?**

One of the main answers is:

```text
/etc/fstab
```

The `/etc/fstab` file describes filesystems and other mountable resources that the system may need to mount automatically or make available for mounting.

In this lab, you will learn how to read and audit `/etc/fstab` safely.

You will:

- inspect the current `/etc/fstab`;
- understand every field in an entry;
- distinguish UUIDs, labels, partition UUIDs, and device paths;
- interpret common mount options;
- understand the final two numeric fields;
- correlate `/etc/fstab` with actual storage and mounted filesystems;
- validate the file without mounting new storage;
- recognize configuration mistakes before they become boot or availability problems.

This is deliberately a **safe inspection and validation lab**.

You will **not** modify the live `/etc/fstab`.

---

## 2. Safety Boundary

`/etc/fstab` is a system configuration file.

A bad entry can cause mount failures and, depending on the system and configuration, can interfere with normal startup or leave expected storage unavailable.

For this lab:

> **Do not edit `/etc/fstab`.**

You will not:

- create or delete partitions;
- format filesystems;
- change filesystem UUIDs;
- modify existing `/etc/fstab` entries;
- add experimental devices to the live file;
- run `mount -a` against a modified live configuration;
- reboot the machine to test an entry;
- use an unknown block device;
- attempt to work around storage errors with destructive commands.

First, identify your environment:

```bash
lsblk -f
```

Then inspect current filesystem usage:

```bash
df -hT
```

These commands should already be familiar from earlier labs.

> **Engineer Note**
>
> Your machine may look different from examples you see elsewhere. A physical server, virtual machine, cloud instance, WSL environment, and container can expose very different storage layouts.
>
> Work with the system you actually have.

---

## 3. What `/etc/fstab` Does

The name `fstab` comes from **filesystem table**.

Its traditional location is:

```text
/etc/fstab
```

Display it:

```bash
cat /etc/fstab
```

A simple system might contain entries resembling:

```text
UUID=7c3a...   /       ext4    defaults    0  1
UUID=A12B...  /boot   ext4    defaults    0  2
```

Your system may contain completely different entries.

It may use:

```text
UUID=
```

or:

```text
PARTUUID=
```

or filesystem labels, device paths, swap entries, network filesystems, special filesystems, or other configurations.

Some cloud or container environments may have a very small `/etc/fstab`, and some may contain almost no active entries.

### Comments

Lines beginning with:

```text
#
```

are comments.

For example:

```text
# /etc/fstab: static file system information
```

Comments are ignored as mount definitions.

Blank lines are also ignored.

You can display lines that do not begin with `#`:

```bash
grep -v '^#' /etc/fstab
```

This may still show blank lines, which is fine.

---

## 4. Understanding the Six Fields

A normal `/etc/fstab` filesystem entry contains six fields separated by whitespace.

The general structure is:

```text
<source>  <mount-point>  <filesystem-type>  <options>  <dump>  <pass>
```

For example:

```text
UUID=abcd-1234  /data  ext4  defaults  0  2
```

Think of the fields like this:

| Field | Meaning | Example |
|---|---|---|
| 1 | What should be mounted? | `UUID=abcd-1234` |
| 2 | Where should it appear? | `/data` |
| 3 | What filesystem type is it? | `ext4` |
| 4 | How should it be mounted? | `defaults` |
| 5 | Should the traditional `dump` utility consider it? | `0` |
| 6 | What filesystem-check ordering applies? | `2` |

The position of each field matters.

Consider:

```text
UUID=abcd-1234  /srv/data  ext4  defaults  0  2
```

Linux interprets it as:

```text
Source          → UUID=abcd-1234
Mount point     → /srv/data
Filesystem type → ext4
Options         → defaults
Dump field      → 0
Pass field      → 2
```

If fields are missing, misplaced, or incorrect, the entry may not behave as intended.

---

## 5. Field 1 — Filesystem Source

The first field identifies **what Linux should mount**.

Several forms are possible.

### Device path

An entry might use:

```text
/dev/sdb1
```

For example:

```text
/dev/sdb1  /data  ext4  defaults  0  2
```

This directly names a device node.

The problem is that names such as:

```text
/dev/sdb1
```

are not always the best persistent identity for storage. Device naming can depend on how storage is discovered.

That is one reason persistent identifiers are commonly used.

### Filesystem UUID

A common form is:

```text
UUID=<filesystem-uuid>
```

Inspect filesystem information:

```bash
lsblk -f
```

Look for the `UUID` column.

You may see something conceptually similar to:

```text
NAME   FSTYPE  LABEL  UUID
sda
└─sda1 ext4           12345678-abcd-....
```

The exact values depend on your machine.

A corresponding `/etc/fstab` source could be:

```text
UUID=12345678-abcd-...
```

A UUID usually provides a more stable filesystem identity than a discovery-dependent name such as `/dev/sdb1`.

However, remember:

> A filesystem UUID belongs to the filesystem. Recreating or reformatting that filesystem can result in a different UUID.

### Filesystem label

A filesystem may also have a label:

```text
LABEL=DATA
```

Labels are easier for humans to recognize, but administrators should manage them carefully because labels are not inherently guaranteed to be unique across all attached storage.

### Partition UUID

You may encounter:

```text
PARTUUID=<value>
```

Do not confuse:

```text
UUID=
```

with:

```text
PARTUUID=
```

They identify different layers.

A useful mental model is:

```text
PARTUUID → identifies a partition

UUID     → commonly identifies a filesystem
```

Depending on the system, you can inspect available identifiers with:

```bash
lsblk -f
```

and, where permitted:

```bash
blkid
```

`blkid` output and visibility can vary depending on permissions and the environment.

### Special sources

Not every `/etc/fstab` source represents a normal disk partition.

You may encounter special sources associated with:

- swap;
- `tmpfs`;
- network filesystems;
- bind mounts;
- other Linux storage arrangements.

Those configurations become more important in later administration work.

---

## 6. Fields 2 and 3 — Mount Point and Filesystem Type

### Field 2: Mount point

The second field specifies **where the filesystem should appear in the Linux directory tree**.

Example:

```text
UUID=abcd-1234  /data  ext4  defaults  0  2
```

The mount point is:

```text
/data
```

After a successful mount, the filesystem becomes accessible through that path.

From JUN-040, remember:

> A mount point is a directory used as the attachment location for a filesystem.

The directory itself and the filesystem being mounted there are different things.

### Field 3: Filesystem type

The third field tells the mount system what filesystem type is expected.

Examples include:

```text
ext4
xfs
vfat
tmpfs
swap
auto
```

You can inspect currently mounted filesystem types with:

```bash
df -hT
```

You can inspect filesystem metadata associated with block devices using:

```bash
lsblk -f
```

Suppose an entry says:

```text
UUID=abcd-1234  /data  ext4  defaults  0  2
```

but the identified filesystem is actually XFS.

That mismatch is something an administrator must investigate rather than blindly forcing the mount.

### Storage-layer mental model

Keep the layers separate:

```text
Disk
  ↓
Partition
  ↓
Filesystem
  ↓
Mount configuration
  ↓
Mount point
  ↓
Files and directories
```

Not every system uses every layer exactly this way. LVM, encryption, RAID, containers, network storage, and other technologies can add more layers.

But the model is useful for junior Linux troubleshooting.

---

## 7. Field 4 — Mount Options

The fourth field contains mount options.

Multiple options are separated by commas.

For example:

```text
defaults
```

or:

```text
ro,noauto
```

Some common options you should recognize at this level are:

| Option | Meaning |
|---|---|
| `defaults` | Use the normal default mount option set |
| `ro` | Mount read-only |
| `rw` | Mount read-write |
| `noauto` | Do not automatically mount through normal automatic processing such as `mount -a` |
| `nofail` | Allow the system to continue when the device cannot be mounted rather than treating that mount as a required successful dependency |

### `defaults`

You will frequently see:

```text
defaults
```

On Linux, this represents the conventional default option set used by `mount`.

For normal Linux filesystems this commonly corresponds to behavior associated with options such as:

```text
rw,suid,dev,exec,auto,nouser,async
```

You do not need to memorize that list for this lab.

What matters now is understanding that:

```text
defaults
```

does **not** mean:

```text
no configuration
```

It requests the normal default behavior.

### `ro`

Example:

```text
UUID=abcd-1234  /archive  ext4  ro  0  2
```

`ro` means:

```text
read-only
```

The filesystem is intended to be mounted without normal write access.

### `rw`

`rw` means:

```text
read-write
```

### `noauto`

Consider:

```text
UUID=abcd-1234  /archive  ext4  noauto  0  2
```

`noauto` prevents that entry from being mounted by normal automatic `fstab` processing such as:

```bash
mount -a
```

The filesystem can still be mounted manually when appropriate.

### `nofail`

`nofail` is useful when a filesystem should not be treated as mandatory for successful system startup.

But do not develop this habit:

```text
Mount failed → add nofail
```

That hides the real question.

Ask:

```text
Why did the mount fail?
```

Possible causes include:

- wrong UUID;
- unavailable storage;
- incorrect filesystem type;
- incorrect mount point;
- invalid options;
- damaged filesystem;
- network storage unavailable;
- storage dependency not ready.

`nofail` expresses intended dependency behavior. It is not a universal repair mechanism.

> **Engineer Note**
>
> Production mount options are a security and reliability decision. Options such as `nodev`, `nosuid`, `noexec`, `_netdev`, and systemd-specific options exist, but they are outside the core scope of this first `/etc/fstab` lab.

---

## 8. Fields 5 and 6 — Dump and Filesystem Check Order

The last two fields often look like:

```text
0 2
```

They are easy to ignore, but you should know what they mean.

### Field 5: `fs_freq`

Example:

```text
0
```

This field is historically associated with the `dump` backup utility.

On many modern Linux systems, you will commonly see:

```text
0
```

which means the filesystem is not selected for that traditional `dump` behavior.

Do not interpret this field as a complete modern backup policy.

A value of `0` here does **not** mean:

```text
This filesystem does not need backups.
```

It only relates to this specific historical mechanism.

### Field 6: `fs_passno`

The sixth field controls filesystem-check ordering associated with boot-time filesystem checking.

Typical values are:

```text
0
1
2
```

Conceptually:

| Value | Meaning |
|---|---|
| `0` | Do not schedule this filesystem through this `fsck` pass mechanism |
| `1` | Highest priority; traditionally used for the root filesystem |
| `2` | Lower-priority checks for other appropriate filesystems |

You may therefore encounter:

```text
UUID=...  /      ext4  defaults  0  1
UUID=...  /home  ext4  defaults  0  2
```

Do not turn this into a rule that every non-root filesystem must use `2`.

Filesystem technologies differ.

For example, XFS does not use the traditional `fsck` workflow in the same way as ext-family filesystems.

The correct value depends on the filesystem and system design.

---

## 9. Audit the Current System

Now connect `/etc/fstab` to the system that is actually running.

Start with:

```bash
cat /etc/fstab
```

Then inspect block devices:

```bash
lsblk -f
```

Inspect currently mounted filesystems:

```bash
findmnt
```

Inspect capacity and filesystem types:

```bash
df -hT
```

Where useful and permitted:

```bash
blkid
```

You are answering different questions with each tool.

| Tool / File | Main Question |
|---|---|
| `/etc/fstab` | What persistent/static mount configuration has been declared? |
| `lsblk -f` | What block devices and filesystem metadata can I see? |
| `blkid` | What filesystem/partition identifiers can be discovered? |
| `findmnt` | What is mounted, and how is it represented in the mount hierarchy? |
| `df -hT` | What mounted filesystem capacity and filesystem types are visible? |

These sources are related, but they are not interchangeable.

### Important distinction

An `/etc/fstab` entry does not automatically prove that a filesystem is currently mounted.

Likewise, a currently mounted filesystem does not necessarily prove that it came from `/etc/fstab`.

A filesystem could have been mounted manually or created dynamically by another part of the system.

This is why administrators correlate evidence instead of trusting one command.

### Inspect a particular path

For example:

```bash
findmnt -T /
```

Then:

```bash
df -hT /
```

And:

```bash
lsblk -f
```

Try to trace:

```text
Path
  ↓
Mounted filesystem
  ↓
Filesystem type
  ↓
Filesystem identifier
  ↓
Underlying storage, where applicable
```

On some systems, particularly containers and layered environments, that mapping may not resemble a traditional disk → partition → filesystem layout.

Document what the system actually shows.

---

## 10. Validate `/etc/fstab` Safely

Reading an `/etc/fstab` entry and thinking it looks correct is not enough.

Linux provides tools that can help validate the configuration.

First create a lab workspace:

```bash
mkdir -p ~/linux-world-labs/JUN-041
cd ~/linux-world-labs/JUN-041
```

Make a copy of the current file:

```bash
cp /etc/fstab fstab.backup
```

Verify:

```bash
ls -l
```

You should see:

```text
fstab.backup
```

This copy is for inspection and evidence only.

You are **not** replacing the system file.

### Verify the live configuration

Run:

```bash
findmnt --verify --verbose
```

On a normal util-linux system, `findmnt --verify` checks the `/etc/fstab` configuration and reports problems or warnings it can detect.

Read the result carefully.

Do not assume that a quiet or successful verification proves that every storage dependency will always mount successfully.

A validator cannot guarantee things such as:

- a removable disk being physically connected tomorrow;
- network storage being reachable during a future boot;
- storage remaining healthy;
- credentials remaining valid;
- every runtime dependency being available.

Think of validation as:

```text
configuration verification
```

not:

```text
proof that future storage can never fail
```

### Why not simply use `mount -a`?

You will frequently see administrators use:

```bash
sudo mount -a
```

after changing `/etc/fstab`.

`mount -a` processes eligible `/etc/fstab` entries and attempts mounts that should be mounted and are not already mounted.

That makes it useful in a controlled administration workflow.

But it is not merely a syntax checker.

It can perform real mount operations.

For this lab, you have not changed the live file, so there is no reason to use `mount -a` as an experiment.

A safer learning sequence is:

```text
Inspect
   ↓
Correlate identifiers
   ↓
Verify configuration
   ↓
Investigate warnings/errors
   ↓
Only then perform controlled mount testing when authorized
```

---

## 11. Troubleshooting `/etc/fstab`

When persistent mounting fails, avoid random edits.

Use an evidence-driven investigation.

### Symptom 1 — Wrong source identifier

Suppose an entry contains:

```text
UUID=1111-2222  /data  ext4  defaults  0  2
```

but:

```bash
lsblk -f
```

shows no filesystem with that UUID.

The important evidence is the mismatch.

Possible explanations include:

- the device is not attached;
- the filesystem was recreated and received another UUID;
- the UUID was entered incorrectly;
- the expected storage is unavailable.

Do not immediately replace the UUID with a guessed `/dev/sdX` name.

Identify the actual storage first.

---

### Symptom 2 — Wrong mount point

Suppose the configuration references:

```text
/srv/data
```

but the expected directory is missing or the path is incorrect.

The mount configuration and intended directory tree no longer agree.

Investigate the target rather than blindly creating directories until an error disappears.

---

### Symptom 3 — Wrong filesystem type

Suppose `/etc/fstab` says:

```text
ext4
```

while:

```bash
lsblk -f
```

identifies the filesystem as:

```text
xfs
```

That is a configuration mismatch requiring investigation.

Do not format the device to make it match the configuration.

The safe question is:

```text
Which information is supposed to be correct?
```

---

### Symptom 4 — Invalid or inappropriate options

An invalid option can cause a mount operation to fail.

An option can also be syntactically valid but inappropriate for the intended workload.

Separate:

```text
Can Linux understand this option?
```

from:

```text
Is this the correct operational policy?
```

Those are different questions.

---

### Symptom 5 — `/etc/fstab` looks correct, but the filesystem is not mounted

Check:

```bash
findmnt
```

Then inspect the relevant entry.

Look for options such as:

```text
noauto
```

Also determine whether the storage itself is available.

Remember:

```text
Configured ≠ currently mounted
```

---

### Symptom 6 — Filesystem is mounted but not present in `/etc/fstab`

This is possible.

It may have been:

- mounted manually;
- mounted by another service;
- dynamically created;
- managed by a container/runtime environment;
- mounted through another system mechanism.

Again:

```text
Mounted ≠ necessarily declared in /etc/fstab
```

---

### A Professional Investigation Flow

When investigating an `/etc/fstab` issue, work through:

```text
1. What is the symptom?
2. Which mount point is affected?
3. What does /etc/fstab declare?
4. What source identifier does it use?
5. Does that source exist?
6. What filesystem type is actually present?
7. What does findmnt show?
8. What does df show?
9. Are the mount options appropriate?
10. Does findmnt --verify report anything?
11. What is the smallest safe correction?
12. How will recovery be verified?
```

This is much safer than randomly changing UUIDs, filesystem types, or mount options.

### Common Mistakes

Avoid these habits:

- copying an `/etc/fstab` line from another server without checking identifiers;
- assuming `/dev/sdb1` always refers to the same intended storage;
- confusing UUID with PARTUUID;
- using `nofail` simply to suppress a problem;
- changing multiple fields at once;
- rebooting immediately after an unverified edit;
- formatting a filesystem because the declared type is wrong;
- assuming a successful `findmnt --verify` guarantees the physical device will always be available;
- treating `/etc/fstab` as a list of currently mounted filesystems.

---

## 12. Challenge Mode — Audit Persistent Mount Configuration

You are now responsible for auditing the persistent filesystem configuration of your current Linux environment.

Do **not** modify the live configuration.

### Step 1 — Preserve a reference copy

```bash
cd ~/linux-world-labs/JUN-041
cp /etc/fstab fstab.backup
```

Verify:

```bash
ls -l
```

---

### Step 2 — Inspect the configuration

```bash
cat /etc/fstab
```

For every active entry you can identify, determine:

1. source;
2. mount point;
3. filesystem type;
4. mount options;
5. dump field;
6. filesystem-check field.

If your `/etc/fstab` contains no meaningful disk entries, that is still valid evidence about your environment.

Do not invent entries.

---

### Step 3 — Correlate storage identifiers

Run:

```bash
lsblk -f
```

Where permitted:

```bash
blkid
```

Determine whether `/etc/fstab` uses:

```text
UUID=
LABEL=
PARTUUID=
/dev/...
```

or another source type.

Do not assume that every `/etc/fstab` entry must appear in `lsblk`. Special and network-backed resources can follow different models.

---

### Step 4 — Compare configuration with runtime state

Run:

```bash
findmnt
```

Then:

```bash
df -hT
```

For the root filesystem, inspect:

```bash
findmnt -T /
```

and:

```bash
df -hT /
```

Answer:

```text
What backs /?
What filesystem type is it?
What is its current mount point?
Can I correlate it with /etc/fstab?
Can I correlate it with lsblk?
```

If the answer to one of those questions is no, explain what your environment shows instead of forcing the expected model.

---

### Step 5 — Validate

Run:

```bash
findmnt --verify --verbose
```

Classify the result as:

```text
No obvious configuration problem reported
```

or:

```text
Warning/error requires investigation
```

If a warning or error appears, investigate it.

Do not modify the live file just to make the message disappear.

---

### Step 6 — Explain these entries

Without adding them to your system, explain what each hypothetical entry is asking Linux to do.

#### Example A

```text
UUID=1111-2222  /data  ext4  defaults  0  2
```

#### Example B

```text
UUID=3333-4444  /archive  ext4  ro,noauto  0  2
```

#### Example C

```text
LABEL=REPORTS  /reports  xfs  defaults,nofail  0  0
```

Your explanation should identify all six fields and the expected mount behavior.

Then identify what you would verify **before** placing any of those entries on a real production system.

---

## Knowledge Check

1. What is the purpose of `/etc/fstab`?

2. Does an entry in `/etc/fstab` prove that a filesystem is currently mounted?

3. Does a mounted filesystem have to appear in `/etc/fstab`?

4. What are the six fields of a normal `/etc/fstab` entry?

5. What is the difference between the filesystem source and the mount point?

6. Why can `UUID=` be preferable to a path such as `/dev/sdb1`?

7. What is the difference between `UUID=` and `PARTUUID=`?

8. What does the filesystem-type field describe?

9. What is the purpose of `noauto`?

10. Why should `nofail` not be used merely to hide mount problems?

11. What does the fifth field traditionally control?

12. What is the purpose of the sixth field?

13. Why should you compare `/etc/fstab` with `lsblk -f`, `findmnt`, and `df -hT`?

14. What does `findmnt --verify --verbose` help you determine?

15. Why is rebooting immediately after an unverified `/etc/fstab` change a poor operational practice?

---

## Interview Prep

### 1. What is `/etc/fstab`?

`/etc/fstab` is the filesystem table used to describe filesystems and other mountable resources, their mount points, filesystem types, options, and related mount/checking behavior.

### 2. Why would you use a UUID in `/etc/fstab` instead of `/dev/sdb1`?

A filesystem UUID provides a persistent filesystem identity that is generally less dependent on device discovery order than names such as `/dev/sdb1`.

### 3. What is the difference between `mount` and `/etc/fstab`?

`mount` performs or manages mount operations, while `/etc/fstab` describes persistent/static mount configuration that mounting tools and the system can use.

### 4. How would you investigate an `/etc/fstab` mount failure?

I would inspect the entry, verify the source identifier and filesystem type with tools such as `lsblk -f` and `blkid`, inspect runtime mounts with `findmnt`, compare capacity/filesystem information with `df`, verify the configuration, and investigate the specific error before changing anything.

### 5. What does `noauto` do?

It prevents the entry from being automatically mounted through normal automatic `fstab` processing such as `mount -a`, while still allowing appropriate manual mounting.

### 6. What does `nofail` do?

It indicates that failure to mount the resource should not be treated like failure of a required mount dependency. It is useful for intentionally nonessential storage but should not be used to conceal configuration errors.

### 7. What is the difference between UUID and PARTUUID?

UUID commonly identifies a filesystem, while PARTUUID identifies a partition. They belong to different storage layers.

### 8. How would you reduce the risk of an `/etc/fstab` change?

I would preserve the existing configuration, change only the intended entry, verify identifiers and filesystem types, validate the configuration, perform controlled mount testing where appropriate, verify the resulting mount, and avoid using a reboot as the first test.

---

## Cleanup and Completion

This lab intentionally avoided changing system mount configuration.

Confirm that you did not modify the live file:

```bash
cat /etc/fstab
```

Your lab workspace should contain only the reference copy you created:

```bash
ls -l ~/linux-world-labs/JUN-041
```

When you no longer need it:

```bash
rm ~/linux-world-labs/JUN-041/fstab.backup
```

Then remove the empty lab directory:

```bash
rmdir ~/linux-world-labs/JUN-041
```

You have completed JUN-041 when you can:

- explain the purpose of `/etc/fstab`;
- identify all six fields;
- distinguish source from mount point;
- distinguish UUID from PARTUUID;
- explain why persistent identifiers are useful;
- recognize common filesystem types;
- interpret `defaults`, `ro`, `rw`, `noauto`, and `nofail`;
- explain the final two numeric fields;
- correlate `/etc/fstab` with `lsblk -f`;
- compare configuration with `findmnt`;
- compare runtime filesystem information with `df -hT`;
- use `findmnt --verify --verbose` as a non-destructive configuration check;
- explain why validation is not the same as guaranteed mount success;
- investigate mismatches without guessing;
- explain why `/etc/fstab` changes deserve careful validation before reboot.

### Engineering Principle

> **Never treat `/etc/fstab` as a file you edit until the error disappears. Treat it as persistent storage configuration whose identifiers, mount targets, filesystem types, options, and boot behavior must all agree with the system you actually operate.**

---

**Next Lab:** `JUN-042 — Temporary Files and /tmp`
