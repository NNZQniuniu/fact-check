# fact-check

[English](README.md) | [中文](README.zh-CN.md)

A recursive claim-verification protocol for AI agents. Put it in front of any paragraph, conclusion, or "I checked it" and it will not let your agent mark something verified that it never actually traced to identifiable evidence.

Chinese name: **事实核查** (递归声明核查).

## The three skills

| Skill | Who invokes it | What it does |
|---|---|---|
| `fact-check` | the model and you | The method itself: decompose → chase every ground chain to a terminus → report per claim. Writes no files by default. |
| `fact-check-with-docs` | you, or an orchestrator | The first pass. Produces `核查-<topic>.md`. |
| `fact-check-by-docs` | you, or an orchestrator | Reads that document and continues. Advances only what is unticked, or carries an unresolved diff. |

The last two ship `disable-model-invocation: true` (same convention as `grill-me` / `grill-with-docs`): they write files, so only a human may name them. Neither shell contains any method — the method lives entirely in `fact-check`.

The check document is plain Markdown. Each claim carries a status, below it the ground (where you looked, plus the verbatim text), then one line of `[x] who date`. If you disagree, you write a diff (before / after) and a reason. **A tick means "I opened it and I accept it" — a machine may not tick.** Restating somebody else's words does not count either. Everything unticked is listed by one command, `grep -n "\[ \]" 核查-*.md` — no script needed.

## The problem it solves

Ask an agent "is this true?" and it will verify the two easiest claims, then declare the whole paragraph credible — the unverified remainder silently inherits the credibility of the verified part. Failure modes this protocol is built against:

- Verifies the surface claim and misses the implicit ones (can these two numbers actually be divided? is a correlation being sold as causation? is "most" quantified?)
- Treats memory, another agent's conclusion, or its own previous turn as evidence
- Says "should be", "I recall", "broadly correct" instead of checking
- Verifies that something *exists*, then treats the entire passage as trustworthy
- Produces a verification report with no denominator, so nobody can see what was skipped

## The kernel

1. **Exhaustive decomposition first.** List every verifiable claim, numbered, including implicit ones — and deliver that list *before* verification starts. The list is the denominator; anything not covered needs somewhere visible to live.
2. **Recursive chain.** For every claim, for every ground, chase *its* ground, until a terminus.
3. **Terminus = "identifiable evidence endpoint", not "axiom".** A file's text proves the file says that. A command's output proves that command produced that output in that environment. A page proves the page currently reads that way. These are evidence about the *material*, not evidence about the *world* — and authority statistics are a secondary aggregate whose scope, sample, and time window can all be wrong, so they never earn the top grade.
4. **Report per claim and publish the denominator.** One line per claim, plus a closing count. "All verified" is banned unless coverage is literally 100%.

## What's inside SKILL.md

- **Claim triage**: excluded (opinion, preference, user-supplied premise) / listed but unverifiable (predictions, oughts, values) / listed and verifiable
- **Evidence grades**: A (first-hand) / B (a source records it) / C (secondary aggregate, relay, translation) — never substitutes for a status
- **Four statuses**: `✓ verified` / `✗ falsified` / `? undetermined` / `○ not covered`, with an ASCII fallback (`OK / NO / ? / -`) for terminals that cannot render the glyphs
- **Undetermined reasons**: no evidence found / sources conflict / circular grounds / beyond capability / no permission / no data
- **A worked example** that calibrates decomposition granularity — the one thing no rule can specify

## Install

### Claude Code

```bash
git clone https://github.com/NNZQniuniu/fact-check.git ~/.claude/skills/fact-check
```

### DSH, or anything that scans `~/.agents/skills`

```bash
git clone https://github.com/NNZQniuniu/fact-check.git ~/.agents/skills/fact-check
cp -r ~/.agents/skills/fact-check/skills/* ~/.agents/skills/
```

DSH scans exactly one level (`<root>/<name>/SKILL.md`). All three skills live in this one repository, so the two shells must be copied up to the top of the skills root — the second line above does that.

### The skill name must be kebab-case ASCII

This is the one gotcha worth reading before you rename anything. DSH validates the frontmatter `name` against `/^[a-z0-9]+(?:-[a-z0-9]+)*$/` at discovery time and drops the file with `ignored: invalid skill name` — it never reaches the registry, and the skill tool then reports `invalid skill name` before it even looks for the skill. A Chinese `name:` therefore makes the skill invisible, no matter what the directory is called.

That is why the frontmatter here says `name: fact-check` while the content is Chinese. **The directory name is irrelevant** — discovery reads the frontmatter; a folder named `事实核查` works fine as long as `name:` is kebab-case.

## Usage

Give it a passage and it returns a decomposition, then a status per claim:

```
Input: "某公司去年营收增长 20%,因为用户数翻倍。"

Decomposition: 6 claims
  ① revenue grew (existence)   ② +20% (number)      ③ time window = "last year"
  ④ user count doubled         ⑤ growth caused by it (attribution)
  ⑥ both figures share a comparable basis (implicit premise)

Report:
  claim 1  "revenue grew"            ✓ verified (grade A: 10-K, p.42)
  claim 2  "grew by 20%"             ? undetermined (sources conflict: 18% vs 20%)
  claim 3  "time window last year"   ✓ verified (grade A)
  claim 4  "user count doubled"      ○ not covered (out of scope this pass)
  claim 5  "because user count ..."  ? undetermined (correlation sold as causation)
  claim 6  "same basis"              ? undetermined (no data on the denominator)
  — 6 claims: 2 verified, 3 undetermined, 1 not covered
```

## Run it in a fresh session

Everything this protocol is worth rests on one distinction: the difference between *"I opened that source"* and *"I remember opening it"*. Inside a long conversation that distinction has no physical basis — your agent's own earlier conclusions and the file text it actually just read are the same kind of token in the same window. A `✓` earned at turn 20 gets spent as evidence at turn 400, and nothing in the context marks it as second-hand by then.

§二 of the skill already rules that a model's own previous turn is a claim to be checked, not a ground. Running the check inside the session that produced the claim asks that rule to fight its own context. It loses.

So:

- **Open a new session.** Feed it only the passage, document, or conclusion under review.
- **Do not feed it the history that produced the conclusion.** No "here's what we found so far", no earlier report, no claim list already wearing tick marks. That material is the *object* of the check, never the background for it.
- **One claim set per session.** Three conclusions to audit is three sessions.
- **Better: run the same set two or three times in independent sessions.** The passes catch different errors, and the misses are not interchangeable — the pass that re-derives a number and the pass that asks what that number counts are doing different work. Do not resolve disagreement by majority: an error that survived two passes and fell to one is still an error. Take the union of the findings.
- Continuing an existing document with `fact-check-by-docs` is the case this is designed for: the document carries the state, so the fresh session needs nothing else.

## Design notes

These are the deliberate trade-offs, since they look like omissions otherwise:

- **Exhaustiveness cannot be self-proved.** Proving that no claim was missed would require performing an equally complete second decomposition — which is just as unprovable. So the protocol does not rely on "miss one claim and the whole thing failed" (a failure condition the violator themselves cannot observe). It enforces a *visible denominator* instead.
- **Grades annotate; statuses conclude.** If a grade could stand in for a status, "grade B" would become a new laundering route to "verified". So a non-first-hand terminus must be written as `✓ verified (grade B)`.
- **Circular grounds are a reason for "undetermined", not a fifth terminus** — a terminus has to terminate. (A cites B, B cites A is a termination failure.)
- **Conditional truth is an annotation** (`✓ holds within premise P`), not a separate epistemic state; otherwise the status field mixes results with process.
- **Predictions, oughts, and values are listed but not chased.** They are not undetermined — they are not truth-apt. Reporting them as "undetermined" would be a category error.

## License

MIT — see [LICENSE](LICENSE).
