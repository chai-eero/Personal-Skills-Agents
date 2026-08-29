# Humanizer rules for PR text

Adapted from the "humanizer" skill (github.com/blader/humanizer, MIT). Apply to
the PR title, commit message, and every prose section of the body. Goal: text
that reads like an engineer wrote it, not a model. Look for **clusters** of these
tells, not a single one. Polish is not the enemy; padding is.

## Hard rules

- **No em dashes (`—`) or en dashes (`–`).** Replace with a period, comma, colon,
  or parentheses. Scan the final draft for both characters before posting.
  (A plain hyphen `-` as a dash, e.g. `160 -> 320` or `duration 10000 ms - bitmap`,
  is idiomatic in eero PRs and fine.)
- **Straight quotes**, not curly `" "` `' '`.
- **Sentence case** for headings and prose. Don't Title-Case Section Bodies.
- **No emojis** in prose (repo checklist templates may ship a 🔒 — keep those as-is).

## Words and phrases to cut

AI vocabulary: delve, leverage, robust, seamless, comprehensive, intricate,
tapestry, testament, underscore(s), highlight(s) (as verb), showcase, foster,
garner, pivotal, crucial, vital, vibrant, enhance, align with, interplay.

Significance inflation: "plays a key/pivotal/crucial role", "underscores the
importance of", "stands as a testament to", "represents a shift", "setting the
stage for", "marks a turning point".

Filler / hedging openers: "It is important to note that", "It is worth noting",
"In order to" (→ "to"), "Due to the fact that" (→ "because"), "At this point in
time" (→ "now"), "has the ability to" (→ "can"), "This PR aims to", "This change
seeks to". Just state what it does.

Chatbot artifacts: "Certainly!", "Of course!", "I hope this helps", "Let me know
if...", "Here is a...". None of these belong in a PR.

Signposting: "Let's dive in", "Let's break this down", "Here's what you need to
know", "Without further ado".

Authority tropes: "The real question is", "At its core", "In reality", "What
really matters", "Fundamentally".

## Structural tells to avoid

- **Negative parallelism:** "Not only X but also Y", "It's not just X, it's Y".
- **Rule of three:** forcing three parallel items when two or four is the truth.
- **Elegant variation:** cycling synonyms for the same thing (channel / frequency
  / band) when one word is clearer. Pick a term and reuse it.
- **False ranges:** "from X to Y" when X and Y aren't endpoints of a real scale.
- **Passive/subjectless fragments:** name the actor. "The ACS node selects..."
  beats "The best channel is selected...".
- **Diff narration:** describe the resulting behavior, not the sequence of edits
  you made. "When AWGN exceeds the threshold, a n/w-wide trigger is sent" beats
  "I added a function that sends a trigger and then called it from the handler".
- **Manufactured punchlines:** stacked short fragments for drama. Not here.
- **Generic upbeat conclusions:** end on a concrete fact or nothing, not "This
  makes the system more robust and reliable."

## The register these PRs actually use

Terse, technical, slightly abbreviated. Real merged eero PRs write: `n/w-wide`,
`b/w`, `intf.`, `chan`, `chan_width`, `ACS node`, `CSA`, `160 -> 320`. Match that
when the user does. Sentence fragments in a Test section are fine. The point of
this file is to strip the *AI padding*, not to formalize the voice.
