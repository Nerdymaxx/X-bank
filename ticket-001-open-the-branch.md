# 🎫 X-Bank Work Ticket #001: "Open the Branch"

**From:** Head Office
**To:** New Systems Team
**Priority:** High — the branch opens Monday.

Welcome to X-Bank. We've just leased a Linux server for our new branch. Head office has sent the official blueprint. Your first job: build the branch exactly as specified. Banks do not tolerate "close enough."

## The Blueprint

```
/srv/bank/
├── accounts/
├── vault/
├── transactions/
├── dropbox/
├── reports/
├── bin/
└── logs/
```

## Your Tasks

1. **Build the entire structure.** Every directory, exact spelling.
2. **Prove it:** install `tree` if you don't have it, and run `tree /srv/bank`.
3. **Open the bank's first three customer accounts** — create empty files `ACC1001.txt`, `ACC1002.txt`, `ACC1003.txt` inside `accounts/`, then put the line `OPENED: 2026-07-14` inside each one *without opening a text editor*.
4. **Navigation drill:** `cd` into `transactions/`. From there, in a single command each:
   - (a) print exactly where you are
   - (b) move to `reports/` using a **relative** path
   - (c) jump to your home directory
   - (d) return to `reports/` using an **absolute** path
5. **Submit your evidence:** `tree /srv/bank` output plus your command history saved to a file called `ticket001_evidence.txt` in your home directory.

**Bonus (for the ambitious):** rebuild the entire blueprint from scratch in **one single command**.

## Acceptance Criteria

- Structure matches the blueprint character-for-character.
- Account files contain the opening line.
- explain, the difference between the path you used in 4(b) and the one in 4(d)

---


