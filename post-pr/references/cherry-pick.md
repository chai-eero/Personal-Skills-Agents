# Cherry-picking PRs to release branches

## Mechanics: use the cherry-pick-pr agent

The `cherry-pick-pr` agent (in `~/.claude/agents/`) owns the full workflow:
gather PR info, create the branch, cherry-pick the squash commit, resolve
conflicts, run tests, push, open the new PR, relabel the original, and optionally
draft a `#node-pr-approval` Slack form. Launch it with:

- `$1` = source PR URL (must be **merged**)
- `$2` = target branch

This skill's job is the **write-up** (title + body) and choosing the right target
branch(es) from the branch map below.

## Cherry-pick PR title

Original title with the target branch appended:
`[CONN-71206] Add ... for v7.16.0-miami` or `... - v7.16.0`. Match the suffix
style the repo's existing cherry-picks use (`for <branch>` vs ` - <branch>`).

## Cherry-pick PR body

Two observed shapes. Both put the cherry-pick provenance in **Notes**.

**Minimal (small/mechanical port, e.g. hostapd #2507):** keep the original
Description/Solution/Test, and in Notes list the sibling PRs across every branch
so reviewers can track the fan-out:

```
## Notes
- Cherry-pick of CONN-71206 to v7.16.0-miami (from eero-ath1250cs)
- nodelib PR: https://github.com/eero-inc/nodelib/pull/13086
- eero-ath1250cs PR: https://github.com/eero-inc/hostapd/pull/2458
- main PR: https://github.com/eero-inc/hostapd/pull/2457
- eero-ath1300csu2 PR: https://github.com/eero-inc/hostapd/pull/2409
```

**nodelib style (e.g. #13205):** full Description/Solution, plus the checklist,
and Notes with `Cherry-pick of <url> for <branch>` and the protobuf PR:

```
## Notes
- Cherry-pick of https://github.com/eero-inc/nodelib/pull/13160 for v7.16.0
- Protobuf PR: https://github.com/eero-inc/protobuf-apis/pull/1197
```

The agent's default body opens with `This PR cherry-picks changes from the PR
<url>.` followed by `---` and the original description. Either that or the
provenance-in-Notes style is acceptable; match the repo's recent cherry-picks.

## Test on a cherry-pick

- Often **`TBD.`** if the change was already validated on the source branch and
  the port is mechanical. #13205 used `Test\nTBD.`.
- If the target branch differs enough that re-testing matters, ask the user for
  fresh logs and follow the normal Test conventions.
- Same draft rule applies: a draft cherry-pick's Test is `TBD.`

## GenAI Disclosure on a cherry-pick

Usually `N/A` for a mechanical port (that's what #13205 used).

---

## eeroOS branch map (Hardware)

Source of truth (may drift — verify against the wiki when precision matters):
https://eeroinc.atlassian.net/wiki/spaces/NODE/pages/3167551489/Hardware+-+eeroOS+Branch+Map

Use this to turn a **hardware platform name** into the branch to target. When the
user says "cherry-pick to Miami" or "to Jupiter", resolve it here.

| Model(s) | Chipset family | ath-eero branch | hostapd branch |
|---|---|---|---|
| Unico, Piccino, Cento | Dakota IPQ4019 | compat-wireless/main-legacy | hostapd/main |
| Eden | Oak IPQ8174 / Hawkeye IPQ807x | ath-eero/eero-main | |
| Andytown G/L, Kilimanjaro | Cypress IPQ6018 | ath-eero/eero-main | |
| Trieste, Firefly, Hornbill, Ladro | Maple IPQ5018 | ath-eero/eero-maple-main | |
| Crane (Bells) | Bells/Alder IPQ95xx | | |
| **Jupiter (Alder)** | Alder IPQ9574 | ath-eero/eero-ath1300csu2 | **hostapd/eero-ath1300csu2** |
| **Merci, Snowbird, Novo** | **Miami** IPQ53xx | (ath1300csu2 family) | (eero-ath1300csu2 family) |
| **Patria** | ath12.5 | ath-eero/eero-ath1250cs | **hostapd/eero-ath1250cs** |

Release branches (e.g. `v7.16.0`, `v7.16.0-miami`) are cut per release and are
what most cherry-picks target. The platform dev branches above (`eero-ath1300csu2`,
`eero-ath1250cs`) are where the main PR usually lands first; releases branch off
those.

Notes from the wiki:
- **auto-pr-ci** (https://github.com/eero-inc/auto-pr-ci) auto-cherry-picks
  across branches. If it covers the target, a manual cherry-pick may be redundant
  — check first.
- FW branch mapping spreadsheet is linked from the wiki page for firmware
  targets.

When a change spans 6GHz platforms, the common fan-out is: land on `main` +
the platform dev branch, then cherry-pick to each active release branch
(`v7.16.0`, `v7.16.0-miami`, ...). The Notes block should list all sibling PRs
so reviewers can see the full set (see hostapd #2507 above).
