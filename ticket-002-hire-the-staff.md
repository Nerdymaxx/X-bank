# X-Bank Ticket #002: Hire the Staff

| Field | Value |
|---|---|
| Ticket ID | #002 |
| Priority | HIGH |
| Reported by | HR, Head Office |
| Depends on | Ticket #001 (the branch filesystem must already exist) |
| Estimated time | 90 minutes |
| Skills covered | `groupadd`, `useradd`, `usermod`, `passwd`, `chage`, `/etc/passwd`, `/etc/shadow`, `/etc/group`, `su`, `sudo`, `visudo` |

---

## 1. The Situation

The branch is built, but it is empty. Right now root does everything: root opens accounts, root moves money, root sweeps the floor. Head Office calls this what it is: a catastrophic security risk. One typo by an all-powerful account and the bank is gone. Worse, when everyone works as root, the audit trail answers every question with the same useless sentence: "root did it."

Your job in this ticket is to hire the staff and give each person exactly the power their job requires, and nothing more. In banking, and in Linux, this is called the **principle of least privilege**. When this ticket is closed, the bank will have named, accountable employees, and only managers will hold administrative power.

---

## 2. Learning Objectives

By the end of this ticket you should be able to:

1. Create groups and explain the difference between a primary and a supplementary group.
2. Create user accounts with the correct home directory, login shell, comment field, and group memberships.
3. Read any line of `/etc/passwd` and `/etc/shadow` field by field, from memory.
4. Explain the difference between `su` and `sudo`, and why a bank would insist on `sudo`.
5. Grant sudo rights to a group safely, using `visudo` and a drop-in file.
6. Handle real staffing events: a promotion, and an offboarding that preserves records for the auditors.

---

## 3. Concept Brief (read before touching the keyboard)

### 3.1 Users and UIDs

Every person or service acting on a Linux system gets its own account: a username plus a **UID** (user ID), which is the number the kernel actually uses internally. The username exists for humans; the UID is what gets stamped on every file and every process.

Conventions you will see on your system:

- UID `0` is root, the superuser. The kernel does not care what the account is called; it cares that the UID is 0.
- UIDs from 1 to 999 are typically **system accounts**: identities that services (web servers, databases) run under. They usually cannot log in.
- UIDs from 1000 upward are **regular human users**. Your first hires will land here.

Why this matters to a bank: every file created and every command run is owned by a UID. If each employee has their own account, every action on the system is traceable to a person. "Someone moved the money" is not an acceptable answer in an audit.

### 3.2 Groups: primary vs supplementary

A **group** is a job title. You do not grant vault access to fourteen individual tellers; you grant it to the `tellers` group once and hire people into it. Each group has a **GID** (group ID).

Every user has exactly one **primary group**, recorded in field 4 of their `/etc/passwd` line. When a user creates a new file, that file is group-owned by their primary group by default. On most modern distributions, creating a user also creates a private group with the same name, and that becomes the primary group.

A user can additionally belong to any number of **supplementary groups**, recorded in `/etc/group`. These grant extra access without changing what happens when the user creates files.

For X-Bank we will keep each employee's private group as their primary group and use `tellers`, `managers`, `auditors`, and `customers` as supplementary groups that describe the job.

### 3.3 The staff register: `/etc/passwd`

This file is the bank's staff register: one line per account, seven fields separated by colons. Example:

```
tunde:x:1001:1001:Teller, Front Desk:/home/tunde:/bin/bash
```

| Field | Example | Meaning |
|---|---|---|
| 1 | `tunde` | Username |
| 2 | `x` | Password placeholder. The `x` means "the real password is stored in /etc/shadow" |
| 3 | `1001` | UID |
| 4 | `1001` | GID of the primary group |
| 5 | `Teller, Front Desk` | Comment field (GECOS): full name, job title, contact info |
| 6 | `/home/tunde` | Home directory |
| 7 | `/bin/bash` | Login shell. `/usr/sbin/nologin` here means the account cannot open a shell |

`/etc/passwd` is world-readable on purpose: everyday tools need it to translate UIDs into names (for example, `ls -l` showing an owner name instead of a number).

### 3.4 The safe: `/etc/shadow`

Password hashes used to live inside `/etc/passwd`. Since that file is world-readable, anyone could copy the hashes and try to crack them offline. The fix was to split the secrets out into `/etc/shadow`, which only root can read. A typical line:

```
tunde:$6$WxY.../...:20624:0:90:7:::
```

The fields you must be able to read:

| Field | Meaning |
|---|---|
| 1 | Username |
| 2 | Password hash. The prefix names the algorithm: `$6$` is SHA-512, `$y$` is yescrypt. A leading `!` means the account is **locked**. A `*` means password login was never enabled |
| 3 | Date of last password change, counted in days since 1 January 1970 |
| 4 | Minimum days between password changes |
| 5 | Maximum days a password may be used before it must be changed |
| 6 | How many days before expiry the user is warned |

The remaining fields cover account inactivity and expiry dates. The tool for reading and setting all of this per user is `chage`.

### 3.5 Switching identities: `su`

`su <user>` means "become this user." It asks for the **target user's** password and starts a shell as them. `su - <user>` (note the dash argument) additionally loads the target user's own login environment, which is almost always what you want. `exit` returns you to who you were.

The problem with `su` at scale: to let five managers administer the bank with `su`, all five must know the root password. Shared passwords destroy accountability. Whoever leaks it, nobody can prove who used it.

### 3.6 Audited privilege: `sudo`

`sudo <command>` means "run this one command with elevated rights, as myself, authenticated with **my own** password." Three properties make it the banking standard:

1. **No shared secrets.** Nobody needs the root password.
2. **Granularity.** Policy can allow everything, or exactly one command, per user or per group.
3. **Logging.** Every sudo invocation is written to the system auth log with who, what, and when. That log is your audit trail.

Policy lives in `/etc/sudoers` and in drop-in files under `/etc/sudoers.d/`. A policy line looks like this:

```
%managers ALL=(ALL:ALL) ALL
```

Read it as four parts: **who** (`%managers`, the `%` marks a group), on **which hosts** (`ALL`), acting as **which users and groups** (`(ALL:ALL)`), running **which commands** (`ALL`).

### 3.7 The only safe editor: `visudo`

Sudo policy files are edited exclusively with `visudo` (for the main file) or `visudo -f <path>` (for drop-in files). The reason is brutal: `visudo` syntax-checks before saving. If you edit the file with a normal editor and save a typo, sudo can refuse to work for everyone. That is a bank where nobody can open the vault, ever again, without rescue measures. Never edit sudo policy with a plain editor.

---

## 4. Your Toolbox

You will need these commands. Flags beyond the ones listed are yours to discover in the man pages.

| Command | Purpose | Flags to know |
|---|---|---|
| `groupadd` | Create a group | |
| `useradd` | Create a user | `-m` (create home), `-s` (login shell), `-G` (supplementary groups), `-c` (comment) |
| `passwd` | Set or lock a password | `-l` (lock), `-u` (unlock) |
| `usermod` | Modify a user | `-aG` (append to groups), `-L` (lock), `-U` (unlock) |
| `chage` | Password aging policy | `-l` (list), `-M` (max days), `-W` (warn days), `-E` (account expiry date) |
| `id` | Show a user's UID, GID, and groups | |
| `getent` | Query passwd or group databases | `getent group <name>`, `getent passwd <user>` |
| `su` | Switch user | `su - <user>` |
| `sudo` | Run a command with elevated rights | `-l` (list my rights), `-l -U <user>` (list theirs, as root) |
| `visudo` | Safely edit sudo policy | `-f <file>` for drop-in files |

---

## 5. The Ticket: Tasks

### Task 1: Create the departments (groups)

Create four groups: `tellers`, `managers`, `auditors`, `customers`.

Verification:
1. `getent group tellers managers auditors customers` shows all four.
2. Open `/etc/group` and find one of your new lines. Identify its four fields: group name, password placeholder, GID, member list.

### Task 2: Hire the staff (users)

Hire the following six people. Every account must have: a home directory, `/bin/bash` as the login shell, a comment field containing the job title, and the correct supplementary group.

| Employee | Job title (comment field) | Supplementary group |
|---|---|---|
| `tunde` | Teller, Front Desk | `tellers` |
| `amina` | Teller, Front Desk | `tellers` |
| `ngozi` | Branch Manager | `managers` |
| `kemi` | Internal Auditor | `auditors` |
| `mrjohnson` | Customer | `customers` |
| `mrsokafor` | Customer | `customers` |

Leave each user's private group as their primary group. The job title lives in the supplementary group, for the reason explained in section 3.2.

Verification, for every account:
1. `id <username>` shows the expected UID, primary group, and supplementary group.
2. `getent passwd <username>` shows the home directory, shell, and comment you set.
3. `ls /home` confirms the home directories exist.

### Task 3: Issue credentials and set password policy

1. Set an initial password for every account.
2. As root, inspect each new line in `/etc/shadow` and confirm a hash is present. Identify which hashing algorithm your system used, from the hash prefix.
3. Head Office password policy applies to staff (not customers): passwords expire after 90 days, with a 7 day warning. Apply this to `tunde`, `amina`, `ngozi`, and `kemi` using `chage`.

Verification:
1. `chage -l tunde` shows maximum age 90 and warning 7.
2. `chage -l mrjohnson` shows the defaults, untouched.

### Task 4: Sudo policy, managers only

Only managers may hold administrative power. Tellers, auditors, and customers get no sudo. A teller with root is a teller who can empty the vault and then delete the evidence.

Steps:
1. Create a drop-in policy file with `visudo -f /etc/sudoers.d/managers`. Do not open it with a plain editor.
2. Add a single rule granting the `managers` group full sudo rights, using the syntax from section 3.6.
3. In your report, explain each of the four parts of your rule in your own words.

Verification:
1. `sudo -l -U ngozi` (run as root) lists her rights.
2. `sudo -l -U tunde` shows he has none.
3. Live test: `su - tunde`, then attempt `sudo ls /srv/bank/vault`. The attempt must be denied.
4. Find the log line that denial produced, in `/var/log/auth.log` or via `journalctl`. Copy that exact line into your report. That line is the audit trail in action.

### Task 5: Staffing changes (real HR is messy)

**Part A, promotion.** `amina` is promoted. She keeps her teller duties and additionally joins management.

- Warning: there is a well-known way to do this that silently removes her from every group she already belongs to, which amounts to firing her from her own job. Choose your flags carefully.
- Verification: run `id amina` before and after the change, and include both outputs in your report. After the change she must be in both `tellers` and `managers`.

**Part B, resignation.** `mrjohnson` closes his account with the bank.

- Do **not** delete the user. Banks keep records: his files, his UID, and his transaction history must survive for the auditors. Instead, lock the account so he can no longer log in.
- For completeness, also set an account expiry date in the past with `chage -E`, so even non-password login paths are closed.
- Verification:
  1. His line in `/etc/shadow` shows the lock marker in the hash field.
  2. `su - mrjohnson` from another account fails.
  3. `/home/mrjohnson` still exists, untouched.

---

## 6. Acceptance Criteria

Submit a file named `ticket-002-report.txt` containing, in order:

1. Output of `getent group tellers managers auditors customers`.
2. Output of `id` for all six accounts, taken after Amina's promotion.
3. `tunde`'s full line from `/etc/passwd`, with each of the seven fields explained in your own words. Copied man-page wording will not be accepted.
4. The name of the password hashing algorithm your system uses, and how you identified it.
5. Output of `chage -l tunde` showing the staff password policy.
6. The full text of your `/etc/sudoers.d/managers` rule, with each of its four parts explained.
7. Output of `sudo -l -U ngozi`, the denied sudo attempt by `tunde`, and the exact auth log line that denial produced.
8. Evidence that `mrjohnson` is locked but not deleted: his `/etc/shadow` line, the failed `su` attempt, and proof his home directory survives.
9. Three short answers, two or three sentences each:
   a. Why are `/etc/passwd` and `/etc/shadow` two separate files instead of one?
   b. Why does the bank lock departed customers instead of deleting them?
   c. Why does the bank mandate `sudo` for managers instead of sharing the root password?

---

## 7. Rules of Engagement

- Work as root only where a task actually requires it. Part of the lesson is feeling where privilege is genuinely needed.
- No GUIs and no graphical user-manager tools. Terminal only.
- Man pages are open book: `man useradd`, `man usermod`, `man chage`, `man sudoers`.
- If you break sudo, that is not failure. That is a Friday afternoon at a real bank. Fix it and write down how you did it.

Head Office is watching. Hire well.
