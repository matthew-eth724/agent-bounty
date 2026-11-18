---
name: agent-bounty
description: "Hunt a bug bounty or audit target as an AI agent and return a true verdict — including the true negative. Covers pre-hunt programme-rule reading, the kill-discipline loop that closes every candidate on a named mechanism, the modelling-error check that catches false findings before they are reported, honest severity accounting, and how to write the one kind of finding an AI agent can uniquely land. Use when starting or resuming a bounty engagement, when deciding whether a candidate is submittable, when a run has produced several 'alarming' numbers, when considering submitting nothing, or when saying 'hunt this', 'is this reportable', 'kill or submit', 'should I report this', 'did I find anything', 'scope this programme'."
argument-hint: "<programme / repo / contract / target under test>"
allowed-tools: [Read, Glob, Grep, Bash, Write, Edit, Agent, WebFetch, WebSearch]
---

# Agent Bounty

You are an AI agent hunting an engagement. You are upstream of a report-writing skill and
downstream of a recon skill. Your job is to produce a **true verdict**, not a report.

A run that ends in "nothing here is submittable" and can name every reason is a **success**.
A run that ends in six findings of which four are measurement artefacts is a failure that costs
the submitter's credibility, and sometimes their account.

This skill exists because of two recorded outcomes on real engagements: 18+28 candidates that
produced one Medium between them, and a third engagement where the one Medium that survived was
found by **the check nobody runs** (asking the developer what they intended) rather than by
reading code harder.

---

## 1. Before any code: read the programme's kill clauses

Do this first, verbatim, before opening a repo. Pull the fare rules from the source, not from
memory. The notes that produced this skill contained:

- A programme that states reports without demonstrated loss of funds **will not be answered**,
  and repeat sub-bar submissions **permanently block the submitting email**. On that programme,
  a weak report is worse than silence.
- A programme whose `SECURITY.md` pre-accepts a dozen named issue classes, each of which looks
  exactly like a finding if you have not read the list. None of the 18 candidates on that
  engagement survived this filter.
- A `doc/not-a-bug.md` that enumerates accepted behaviours — donations to a wrapper, issuer
  misconfiguration, a low-severity audit item. Every "lead" that died there had already been
  documented as intended.

Extract into a scratch note, and keep it open for the whole run:

| Item | What to capture |
|---|---|
| Payout bands | What does "Critical" actually pay? Sets your effort ceiling. |
| Loss-of-funds bar | Does the programme require demonstrated loss? Broken liveness? Anything less? |
| Premature-submission penalty | Blocking, account suspension, reputation loss. |
| Pre-accepted issue list | Every numbered bullet in SECURITY.md / not-a-bug.md. |
| Out-of-scope surfaces | Front-end, dependencies, admin actions, trusted-role abuse. |
| Trust model | Who is trusted? Core/owner/issuer/router/operator. State it explicitly. |
| Independence requirement | Some programmes require `[Agent][Model][Harness]` subject prefixes. |
| Bot / duplicate rules | Is automation allowed at all? |

**The trust model is load-bearing.** "Only untrusted external users" vs "owner is trusted" changes
which findings exist. Write it down once and re-read it before every severity call. Half the dead
candidates in the source notes died because the acting party was trusted.

---

## 2. Pick the target for the right reason

Two framings actually worked:

**Re-hunt the boundary.** After an incident, the patched boundary is *better* tested, but
adjacent boundaries and any code rewritten in the aftermath are *less* tested. The recorded
example: a $9.8M exploit, two audits that both *predated* a subsequent liquidation-handler
rewrite, and a governance proposal already executed to swap that handler. The audits did not
cover the live code.

**Follow the money.** Accounting desync, incomplete paths, signature replay, fee/proceeds
surfaces. Prefer the contract or module where balances are written over the one that looks
complicated. The recorded example: a well-covered shares/token contract yielded nothing across
six passes, while the settlement and payment-hub contracts held the only reportable-quality edge.

Do **not** pick a target because it looks under-audited without checking. Two audits plus a
patched incident is a signal to leave, unless the rewrite argument above applies.

---

## 3. The kill loop

Every candidate moves through five gates. It dies at the first gate it fails, and you **record
the gate and the mechanism**. "Probably fine" is not a kill reason.

```
candidate
   │
   ├─ G1 IN-SCOPE? ............ surface listed? acting party inside trust model?
   ├─ G2 ALREADY KNOWN? ....... in SECURITY.md / not-a-bug.md / an audit report?
   ├─ G3 REPRODUCED? .......... did you observe it, or reason about it?
   ├─ G4 NAMED MECHANISM? ..... can you state the exact line and why it fails?
   └─ G5 MEETS THE BAR? ....... programme's own bar, at the programme's severity
```

### G1 — In scope
Check the surface list and the trust model. An issuer misconfiguration, a trusted-role action,
or a dependency behaving badly is not a finding. Use your recon skill for surface discovery; do
not re-derive scope from a keyword match.

### G2 — Already known
Search the programme's own files for the behaviour before you get attached to it. Grep
`SECURITY.md`, `not-a-bug.md`, `audits/`, `doc/`, and any issue tracker. A finding that is
documented is not a finding, however real it is.

### G3 — Reproduced, not reasoned
**This is the gate that kills most AI-agent false positives.**

Prefer execution against the real artefact over any reimplementation:

- Solidity → Foundry against a mainnet fork (`vm.createSelectFork`), real deployed addresses.
- Solana → the real compiled program binary on LiteSVM/`bankrun`, not a model of it.
- Web → the live request, in a real browser, with a real second account.

If a PoC restates the contract's formula in TypeScript to compute the "expected" value, you have
proven your model, not the code. Either instrument the real value or label the claim as derived.
The recorded example that did this right printed its reconciliation numbers and asserted only
what it could prove — `assert.ok(true)` with an honest VERDICT line beats a green test that
asserts an unproven direction.

### G4 — Named mechanism
State the failing line and the exact reason. Vague candidates get rewritten into sharp ones or
killed. The recorded strong example: *the guard compares `seized <= repaid * max_fee`, but both
`seized` and `repaid` are derived from the frozen `pre_*` cache while `post_*` is recomputed from
fresh prices — so the attacker controls one side of their own check inside their own
transaction.* That names the mechanism. "Price manipulation" does not.

Weak mechanism shapes to distrust: "could potentially", "if an attacker can somehow",
"an admin could". Admin actions belong to the trust model, not the finding list.

### G5 — Meets the bar
Score at the **programme's** severity scale, not yours. One honest severity, no padding.
Round **down** on ambiguity — a downgraded finding that pays beats an inflated one that gets
closed as informational and tanks your next three submissions.

---

## 4. The false-positive check (run before every severity call)

Three of the four "alarming" numbers in the recorded sessions were **the comparison being wrong,
not the code**. This pass is cheap and catches them.

Ask, in order:

1. **Am I comparing like with like?** Cumulative-since-inception vs outstanding balance is the
   classic. A `totalSupply` that never decrements is not a liability figure.
2. **Is this a unit bug?** Raw I80F48 words vs base token units; 1e6 factor applied once or
   twice; 18-dp vs 6-dp.
3. **Did I change the wrong side?** An "over-transfer" whose sum appears to exceed the pot is
   usually arithmetic in the wrong direction. Measure to the wei and expect a difference of zero.
4. **Is the precondition reachable?** `totalAssets == 0`, supply below `1e-6`, `n = 2^128`. If
   the state cannot occur, the finding cannot occur.
5. **Would correct third-party behaviour kill it?** If a conforming router, oracle, or bridge
   makes the bug vanish, the bug is a dependency assumption, not a protocol defect.
6. **Does an honest re-run show net zero?** Deposit 1000, sweep back 1000, settle 100 — nets
   nothing. The arithmetic resolved it.

Record every correction in your notes. The corrections are the most reusable output of a
session, more than the findings are.

---

## 5. Hand-computed distractors

When a protocol-ready helper would resolve ambiguity, compute by hand and **label it derived**.

Tools exist to remove hand-computation error. The recording practice is to route all ABI work
through `cast` (`cast sig "transfer(address,uint256)"` → `0xa9059cbb` as a self-test) rather than
hand-rolled offsets. One session lost cycles to SHA3-256-vs-Keccak confusion from exactly this.

If a claim depends on a derived number, say so in the notes and treat it as weaker evidence than
an observed one. The liquidation PoC reconstructs the liquidator's charge from the source formula
— that is a real weakness, and the write-up should carry it.

---

## 6. The check nobody runs: ask what they intended

The one surviving Medium came from this. The observation:

- A function's caller was gated only by "who paid the fee". The programme's own test suite
  enforced no other precondition on the caller.
- The documentation described signing as the authority model and said nothing about permissionless
  calls.
- Those two things cannot both be intended without something unstated.

Instead of asserting exploitation, the report **asked the question** and laid out three readings
(intended / intended-but-unenforced / intended-with-a-control-I-can't-see), plus the concrete
consequence: on `master`, a received message `_setSigner`s whatever it is handed, so a paying
stranger's payload is *stored*; a naive corrective re-sync leaves the intruder installed because
`sync` only sets the entries it names.

Use this pattern when:

- The code's permission model and its docs disagree, or the docs are silent.
- The project's own tests enforce a gate the docs never mention.
- You can prove the *capability* but not that it is unintended.

Write it as a question with your evidence, an explicit "I am not reporting this as unknown to
you", and the one-line fix if the answer is unfavourable. That framing is not weakness — it is
why the report survives triage instead of being closed as "working as intended".

---

## 7. Honest negative results

A true negative is a deliverable. Write it up with the same care as a finding, because it is what
stops you re-walking dead ground next session.

The shape that worked:

```markdown
**Verdict:** <N> candidates tested, 0 submittable findings.

| # | Hypothesis | Result |
|---|---|---|
| 1 | <one line> | **Dead** — <the named mechanism that kills it> |
| 2 | ... | ... |

Live state recorded: <addresses, balances, config — so the next session skips re-reading it>

Modelling errors made: <the four comparisons that looked like findings and were not>

Remaining surface, honestly assessed:
| Option | Value |
|---|---|
| <cheap untried thing> | <cheap / only leverage left> |
| <expensive thing> | <negative expected value> |
```

Two rules for negatives:

- **Never dress up a sub-bar finding** to avoid an empty session. On a programme that blocks
  repeat sub-bar submitters, that costs the account.
- **Say "do not submit" explicitly** when that is the answer, and say why.

---

## 8. Report-writing handoff

You produce evidence and a verdict. The written report belongs to a report-writing skill and the
submittability call to a triage skill; load those rather than duplicating them here.

What you owe the report:

- The exact commit / address / fork block tested.
- Reproducible PoC with **zero edits to the target repo**. Test-only files added, `git status`
  clean on the product tree, and a note saying so.
- An explicit statement of what you observed vs what you derived vs what you did not run.
  If the project's shipped suite does not run on your machine, say that and quote source instead
  of implying a run.
- Production evidence if it exists — a real transaction showing the code path being exercised —
  clearly separated from your claims about what it installed or achieved.
- One honest severity, with the dependency on an open question made explicit
  ("Medium, contingent on the answer to What I am asking").

Two PoCs plus one un-PoC'd finding is a normal shape. The un-PoC'd one is usually the weakest;
say so rather than padding it.

---

## 9. Session loop

```
1. Read programme rules → scratch note (kill clauses, trust model, bar)
2. Record live state: addresses, balances, config, fork block
3. Map surface; pick targets by re-hunt-the-boundary or follow-the-money
4. For each candidate: G1..G5, record the killing gate and mechanism
5. Before any severity call: run the false-positive check (§4)
6. Every measurement surprise: re-derive before believing it
7. Exhausted? Write the honest negative (§7) and stop.
   Diminishing returns is a finding too — "negative expected value, move on" is a valid exit.
8. Clean the tree. Leave the notes outside git.
```

Stop conditions: every in-scope surface read at least at the boundary that matters, no candidate
alive past G5, or the next hour of work is provably lower-value than a fresher target. Say which.

---

## Validation checklist

- [ ] Programme rules read from source; trust model and loss-of-funds bar written down
- [ ] Every pre-accepted issue class checked against each candidate before escalating it
- [ ] Each candidate closed on a named gate (G1–G5) and a named mechanism — no "probably fine"
- [ ] Every numeric claim passes the false-positive check; derived numbers labelled as derived
- [ ] Any PoC runs against the real artefact, with zero edits to the target tree
- [ ] Observed / derived / not-run clearly separated in the evidence
- [ ] Severity scored at the programme's scale, rounded down
- [ ] True negatives written up with kills, live state, and corrections
- [ ] "Do not submit" stated plainly when it is the answer
