# Repo-specific PR templates

Confirm the live template by looking for `.github/pull_request_template.md` or
`.github/PULL_REQUEST_TEMPLATE/` in the repo before posting. What follows are the
observed conventions from recent merged PRs.

---

## nodelib (eero-inc/nodelib)

Python management code on eero devices. PRs squash-merged to `main`.

- **Default base branch:** `main`
- **Labels:** ACS/wifi PRs carry `wifi-acs`. Add labels the user confirms.
- **Branch naming:** `chai/CONN-64657` style (`<user>/<TICKET>`).

**Body sections (in order):**

```
## Description
## Solution
## GenAI Disclosure
## Test
## Checklist
## Notes
```

**Checklist (copy verbatim, leave unchecked unless confirmed):**

```
- [ ] 🔒 If this PR changes product name used in config, I understand official names can only be used at PVT RTM
- [ ] If this PR introduces a call to a new system command, I have add it to Fenrir's RDEPENDS for nodelib
- [ ] If this PR is a **revert** indicate here what version the reverted commit was introduced:
```
(The revert line is present on cherry-pick/release-branch PRs; the first two
appear on all.)

### Worked example — PR #12849 (feature, real tests)

- Title: `[CONN-64657] Add channel switching and ACS avoidance following AWGN events`
- Description opened by naming the prior work it builds on (CONN-64654) and the
  gap it closes, in two sentences.
- Solution broke the approach into the two mechanisms (immediate CSA, ACS
  avoidance), explained the width-first channel selection tradeoff in plain
  terms, and called out a bug found and fixed along the way.
- GenAI Disclosure: `Used Kiro with multiple iterations.`
- Test: labeled scenarios `Test A`..`Test F`, each a one-line scenario followed
  by fenced real logs, with `← avoided` / `← available` annotations on the
  scored-channel dumps. Ended by noting unit-test coverage.
- Notes: log-message cleanups and a removed escalation path.

---

## hostapd (eero-inc/hostapd)

C. AP daemon. PRs squash-merged, but base branch is a **hardware/release branch,
not main** (see the branch map in `cherry-pick.md`).

- **Default base branch:** the active dev branch for the platform, e.g.
  `eero-ath1300csu2` (Jupiter/Miami), `eero-ath1250cs` (Patria). Confirm with the
  user which platform they built against.
- **Branch naming:** `chai/CONN-64653` style.
- Reviewer lists tend to be large (platform touches many owners) — only add
  reviewers the user names.

**Body sections (in order):**

```
## Description
## Solution
## GenAI Disclosure   (present on feature PRs; omit or "N/A" on tiny ones)
## Test
## Notes
```

hostapd PRs often have **no Checklist section**. Don't add one that isn't in the
template.

### Worked example — PR #2179 (feature, real tests)

- Title: `[CONN-64653] ap: Add AWGN Channel Switch event and configurable switch threshold`
  (note the `ap:` component prefix after the ticket — common in hostapd.)
- Description: one line on what's added, then why nodelib needs it.
- Solution: the eloop timer mechanism and the config/runtime knob, plainly.
- Test: `### Test 1/2/3` scenarios, each a one-line expected-outcome then fenced
  logs labeled by node role (ACS node / Leaf node / detecting node).
- Notes: cross-repo PR links (paired nodelib PR, protobuf-apis PR) and
  `Needs to be cherry picked for other 6GHz branches`.

---

## Cross-repo PR links

nodelib and hostapd changes frequently ship together and reference a
protobuf-apis PR. Put these in **Notes**:

```
## Notes
- nodelib PR: https://github.com/eero-inc/nodelib/pull/XXXXX
- hostapd PR: https://github.com/eero-inc/hostapd/pull/XXXX
- Protobuf PR: https://github.com/eero-inc/protobuf-apis/pull/XXXX
- Needs to be cherry picked for other 6GHz branches
```
