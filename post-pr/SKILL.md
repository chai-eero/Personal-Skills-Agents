---
name: post-pr
description: >-
  Write and open GitHub pull requests for eero repos (nodelib, hostapd, and
  friends) with a consistent, human-sounding title, commit message, and PR
  body. Handles the Description / Solution / GenAI Disclosure / Test / Checklist
  / Notes structure, log-pasting conventions, draft PRs (Test defaults to TBD),
  and cherry-picking a merged PR to release branches using the eeroOS branch
  map. Use when the user says "post a PR", "open a PR", "write up this PR",
  "cherry-pick this PR to <branch>", or is drafting a PR body/commit message.
---

# Post PR

Draft and open a pull request that reads like a human wrote it, following the
eero PR conventions. Also handles cherry-picking a merged PR to one or more
release branches.

Two modes:
1. **New PR** — write the title, commit message, and body for a change, then open it.
2. **Cherry-pick** — port a merged PR to a target branch (see [Cherry-picking](#cherry-picking)).

Figure out which mode from the request. "cherry-pick", "port to <branch>",
"backport" → cherry-pick. Otherwise → new PR.

## Before you write

Gather the facts. Do not invent any of them.

1. **What changed and why.** Read the diff (`git diff`, or `gh pr diff` for an
   existing PR). Get the actual behavior change, not a guess from the branch name.
2. **The ticket.** eero PRs are titled `[CONN-xxxxx] <summary>`. Ask for the
   ticket if it is not obvious from the branch name (branches look like
   `chai/CONN-64657`) or the user's message.
3. **The repo.** Conventions differ slightly per repo — see
   `references/repo-templates.md` for nodelib and hostapd specifics.
4. **Whether it is a draft.** If the user says draft, or the change is not yet
   tested, it is a draft. **Draft PRs default their Test section to `TBD.`** —
   do not fabricate test output.
5. **Test evidence.** For a non-draft PR you need real logs or test results.
   Ask the user to paste them if you do not have them. Never make up log lines.

## Writing style

Write like an engineer typing to their team, not like a model generating a
report. **Read `references/humanizer.md` and apply it to the title, the commit
message, and every prose section of the body.** The short version:

- No em dashes or en dashes. Use a period, comma, colon, or parentheses.
  A plain hyphen `-` used as a dash (`switched 160 -> 320`) is fine and common
  in these PRs; the ban is on `—`/`–`.
- No AI-vocabulary padding: delve, leverage, robust, seamless, comprehensive,
  underscores, highlights the importance, plays a key/pivotal role, testament to.
- Cut filler openers and hedging: "It is important to note that", "In order to",
  "This PR aims to". Just say what it does.
- Plain sentence case for prose. Don't bold phrases for emphasis.
- Describe the change as it is, not as a diff narrative ("we now do X" is fine;
  avoid "I changed the function to also handle Y and then updated the caller").
- It is okay to be terse and slightly informal. Real eero PRs write "n/w-wide",
  "b/w", "intf.", "chan", "ACS node". Match that register when the user does.

Do not overcorrect into stilted prose. The goal is natural, not stripped.

## Title

Format: `[CONN-xxxxx] <concise imperative or noun-phrase summary>`

- Lead with the ticket in square brackets.
- Summary describes the change, not the file. Good:
  `[CONN-64657] Add channel switching and ACS avoidance following AWGN events`.
  Bad: `[CONN-64657] Update awgn.py`.
- No trailing period. Keep it under ~72 chars if you can.
- For a cherry-pick, append the target branch: `... - v7.16.0` or
  `... for v7.16.0-miami` (match whatever the repo's existing cherry-picks use).

## Commit message

**Subject line (commit title):**
- Same as the PR title: `[CONN-xxxxx] <summary>`, imperative mood, no trailing period.
- Keep it under ~72 chars. This is what shows up in `git log --oneline`.
- PRs in these repos are **squash-merged**, so the subject becomes the squash
  commit subject. Keep it clean.

**Extended message (commit body):**
- Blank line after the subject, then the body. Wrap prose at ~72 columns.
- Include a body **only if it adds something the subject doesn't** — the *why*,
  a non-obvious constraint, or a link to the ticket/related PR. A one-line
  subject with no body is fine for a small, self-explanatory change.
- Explain the reasoning, not the diff. "Timer is one-shot so a flapping AWGN
  signal doesn't spam CSA triggers" is useful; "edited awgn.c and hostapd.conf"
  is not.
- Don't paste logs or the full PR body into the commit message. The rich write-up
  (Test evidence, checklist, cross-repo links) lives in the **PR body**; the
  commit body is the short durable "why".
- Apply the same humanizer rules (no em/en dashes, no AI padding).
- For a cherry-pick, the `cherry-pick-pr` agent amends the subject to append the
  target branch and preserves the original commit body — you don't rewrite it.

## PR body

Use this structure (it matches the repo templates — confirm section names
against `references/repo-templates.md` for the specific repo):

```
## Description
<Why this change exists. The problem or the gap. 1-3 sentences. Link the
prior/related work if this builds on it.>

## Solution
<What the change does and the key decisions. Explain non-obvious tradeoffs.
Call out any bug you found and fixed along the way.>

## GenAI Disclosure
<If you used an AI tool, say which and how, e.g. "Used Kiro with multiple
iterations." If none, "N/A". Only include this section if the repo template has it.>

## Test
<See "Test section" below. For draft PRs: exactly "TBD.">

## Checklist
<Copy the repo's checklist verbatim, leave boxes unchecked unless the user
confirms an item applies.>

## Notes
<Cross-repo PR links (protobuf, nodelib<->hostapd), cherry-pick intentions
("Needs to be cherry picked for other 6GHz branches"), follow-ups, caveats.>
```

Omit a section only if the repo's template does not have it. Keep the headings
the template uses.

## Test section

This is the most important section. Reviewers trust it.

- **Draft PR → `TBD.`** Nothing else. Don't invent tests.
- **Real PR:** describe each test as a short labeled scenario, then paste the
  evidence. Structure that works well (from real eero PRs):

  ```
  ### Test 1
  <one line: the scenario and the expected outcome>

  <short label of what node/context the logs are from>
  ```
  <pasted log lines — real, unedited>
  ```
  ```

- Name the scenarios by what they prove: `Test A - Leaf detects AWGN, ACS node
  switches and n/w-wide CSA succeeds`. Cover the happy path, the edge cases, the
  fallback/limit paths, and mention unit tests if they exist.

### Pasting logs

- Always fence logs in triple backticks. For multi-block evidence, use one fence
  per logical block with a one-line label above it (which node, which stage).
- Paste **real** log lines. Trim noise with `...` on its own line between
  relevant lines, but never edit or synthesize a line.
- Keep the lines that carry the signal: the event, the decision, the outcome
  (e.g. the `AP-CSA-FINISHED ... success=1`, the `Switching X -> Y (score=...)`).
- Highlight the meaningful value with a trailing `  ← ...` annotation only when
  it helps a reviewer scan (e.g. `← avoided`, `← selected`). Use sparingly.
- If logs are long, lead with the one or two lines that prove the outcome, then
  the supporting detail.

## Opening the PR

Once the body is drafted:

1. Show the user the full title, commit message, and body for review before
   pushing anything.
2. Push the branch if needed, then open with `gh pr create` (or the GitHub MCP
   `create_pull_request`). Pass `--draft` for a draft PR.
3. Set base branch correctly. For a normal PR that is usually `main` (nodelib)
   or the active dev branch (hostapd, e.g. `eero-ath1300csu2`). For a
   cherry-pick it is the target release branch.
4. Add reviewers / labels only if the user asks or the repo clearly expects them
   (e.g. nodelib ACS PRs carry the `wifi-acs` label).

## Cherry-picking

When the user asks to cherry-pick / backport a PR to a release branch:

- **Delegate to the `cherry-pick-pr` agent** — it owns the full mechanical
  workflow (fetch, branch, cherry-pick the squash commit, resolve conflicts,
  push, open the PR, relabel the original). Launch it with the source PR URL and
  the target branch.
- Your job in this skill is the **write-up** the agent produces the body from,
  and picking the right target branch. Read `references/cherry-pick.md` for:
  - The cherry-pick PR body format (`Cherry-pick of <url> to <branch>`, the
    Notes block listing the sibling PRs on other branches, Test often `TBD.`).
  - The **eeroOS branch map** — which hostapd / nodelib / ath-eero branch goes
    with which hardware platform (Jupiter, Miami, Patria, etc.), so you target
    the right branches.
- If the user names a hardware platform ("cherry-pick to Miami") rather than a
  branch, resolve it to the branch via the branch map in
  `references/cherry-pick.md`.

## Repo specifics

See `references/repo-templates.md` for the exact section list, checklist text,
default base branch, and labels for **nodelib** and **hostapd**, plus worked
examples of real merged PRs to model tone against.
