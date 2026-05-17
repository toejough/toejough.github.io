# Engram, a month later: what I built, what I tore out, and what's left

_April 2026_

> **Draft v2 for Joe.** I kept your inserted prose verbatim and rewrote mine in your voice — shorter declaratives, fewer flourishes, no italic emphasis, no "the pivot came" / "deceptively hard" / "the heart of" style. Dropped the Stop hook section per your note. Added a "How memories work now" section that summarizes the relevant README chunks with a link for depth, per your other note. Remaining `[?]` blocks are narrower now — mostly questions your inline edits answered implicitly that I'd still like to source-check. All other changes flagged at the bottom.

---

## A month ago

[A month ago I wrote about engram](engram) — a Claude Code memory plugin built around a closed feedback loop. Extract lessons from sessions. Surface them via BM25 on every prompt. Track whether each memory got followed, contradicted, or ignored. Classify the corpus into effectiveness quadrants (working, leech, hidden gem, noise). Run a periodic `/memory-triage` pass that proposed fixes for the ones misbehaving. The whole point was: measure impact, diagnose, propose, repeat.

I want to talk briefly about what was dissatisfying the more I used it. In particular, the memory checks were time consuming, and constantly filled the screen with their evaluations. Additionally, for all that noise, I wasn't convinced that the memories were being surfaced well - the keyword matching was just not high quality enough, and all the math and classification I'd added was adding complexity without clear payoff.

What's there now is smaller, simpler, and (at least in my testing) more obviously effective. Four skills — `/prepare`, `/recall`, `/learn`, `/remember` — a lean Go binary, two typed memory shapes, and a rule I settled on late: read from every memory surface Claude Code exposes, write only to the one directory engram owns.

If you want to skip the failed experiments and go straight to what's new, [jump to "How memories work now"](#how-memories-work-now).

Between the March post and today there are about 900 commits on `main`. Most of that is two major features I built and then abandoned, plus the rewrite that replaced them.

---

## What I tried that didn't work

### The `/adapt` skill and adaptive policies

The first thing I tried after the March post was closing the feedback loop better, to try to improve the quality and value of the memories. The March version measured memory effectiveness but didn't act on it — surfacing weights were hardcoded, the extraction prompt was static. I attempted to make those weights dynamic based on lived experience of the individual end user with the memories.

The plan was a `policy.toml` that stored learned adjustments, a maintenance pass that generated proposals ("de-prioritize keyword X, broaden situation on memory Y"), and an `/adapt` skill that walked me through reviewing them.

I built it. It landed. It shipped.

I removed it four days later.

The memory quality was still not improving - engram continued to surface terrible memories. I decided the keyword matching was a dead end. I briefly considered using embeddings, but Haiku calls are nearly free, so I decided to try that route. But going that direction was likely to add even more noise to the Claude interface.

### Multi-agent orchestration via tmux

About this time someone showed me the [mycelium](https://github.com/mycelium-io/mycelium) project. Very cool! Agents talking to eachother without depending on a single lead! I decided to try to take some cues from them for a completely different direction: multi-agent coordination. A lead agent the user talks to, reactive task agents running in tmux panes, a chat file at `~/.local/share/engram/chat/<project-slug>.toml` where they coordinated via a typed message protocol (`learned`, `ready`, `shutdown`, `escalate`). Primarily, I wanted to have a co-agent running that was the engram memory helper: watch activity, surface valuable memories, note new learnings along the way.

I built most of it. Six new skills, four new CLI commands, a chat protocol, a tmux orchestration layer.

I started with tmux controls and interactive claude sessions as the other agents. This was super useful for me to really observe and interject when things were going wrong. It was also a large complexity layer I wanted to drop once I built up confidence in the interactions.

Eventually I did drop the tmux control layer, opting for a server that ran and fired off `claude -p --resume <session-id> ...` commands, but in doing so I lost visibility into what was going on. As I kept building, it was increasingly difficult to understand where the problems were.

At this point, I realized I was spending all my time debugging the chat interactions, and burning tokens like you wouldn't believe. This wasn't getting me closer to my goal, and I decided I did _not_ want to rewrite my own fully fleshed out version of mycelium to get the reliability I wanted. As cool as that project is, I also didn't want engram to depend on it.

---

## Backing up

I decided to back way up. What's obviously been working that I can build on? Two things: the `/recall` skill, which searched our session history for relevant information, and the Haiku memory matching. This led me to what I've got now: mechanical search/refinement/recording of memories via the engram binary, API-based Haiku calls for matching & summarizing, and skill-based evaluation/judgement for interacting with both.

---

## How memories work now

The rewrite kept the pieces that were earning their keep and dropped the rest. The full design lives in [the README](https://github.com/toejough/engram#readme); here's the short version of why the shape is what it is.

### Two types instead of one

Memories split into two shapes:

- **Feedback** — a behavioral correction. Four fields: **S**ituation, **B**ehavior, **I**mpact, **A**ction. Answers "how should I behave here?"
- **Fact** — declarative knowledge as a triple: **S**ubject, **P**redicate, **O**bject. Answers "what's true about this project?"

The old schema was one flat blob that had to pretend to be both. That meant every memory lost some structure or pretended to be a type it wasn't. Splitting them lets `/learn` and `/remember` guide you through the right fields for each kind, and lets retrieval rank them differently.

The split maps onto a long-standing distinction in cognitive psychology between semantic memory (facts) and behavioral rules (procedures you apply in context). SBIA — [Situation, Behavior, Impact, Action](https://pmc.ncbi.nlm.nih.gov/articles/PMC8238687/) — is an existing feedback framework (originally applied to clinical preceptor feedback in medical education); engram just adopts it as-is. SPO is just an [RDF triple](https://en.wikipedia.org/wiki/Semantic_triple); the constraint is that you have to name the entity explicitly, which keeps facts retrievable rather than drifting into paragraphs.

### Situation is the field that matters

Both types share a `situation` field, and it's the one that carries the retrieval signal. It describes the task you'd be doing before the lesson is known — "writing async Go tests", "implementing Claude Code hooks". It's what the recall query matches against.

A situation phrased backwards — "after the race condition in test X" — describes a diagnosis, and won't surface for a fresh attempt at the same kind of code. Phrased forward — "writing async Go tests" — it will. Most of the quality gates in `/learn` and `/remember` exist to catch situations from drifting into hindsight.

### Read everywhere, write only what you own

Engram reads from every memory surface Claude Code exposes — `CLAUDE.md` (with `@`-imports), `.claude/rules/`, Claude Code auto-memory, project + user + plugin skills — and merges them all into one Haiku-ranked retrieval pass. It only writes to its own directory at `~/.local/share/engram/`.

The asymmetry is deliberate. Reading everywhere surfaces knowledge already captured somewhere without asking you to re-enter it. Writing only to engram's own directory means uninstalling engram leaves every other surface untouched. I didn't want engram to muck around with end-users' claude files or skills that they may have very intentionally written.

### The four skills

- **`/prepare`** — fires before new work. Breaks the pending task into 2–3 sub-topic queries and primes Claude with ranked context.
- **`/recall`** — the retrieval engine the other skills build on.
- **`/learn`** — fires at completion boundaries. Filters candidate lessons through three gates (does this recur? is it actionable? is this the right home?) before writing.
- **`/remember`** — `/learn`'s user-triggered sibling. Same gates, explicit approval, marked `source = "human"`.

The agent decides when to load memories. Two hooks (after every user message and tool call) remind Claude to consider calling `/prepare` and `/learn` before and after work, but they don't force anything. Without those hooks, the agent tended to ignore the frontmatter guidance in the skills, no matter what I put in there.

Full details including the recall pipeline, the gates, the auto-memory integration, and the psychology references are in [the README](https://github.com/toejough/engram#readme).

---

## Migration

If you ran engram before commit `cfd5fb5` (April 17), the memory file layout and TOML schema both changed. Do not start a new session until you run `/migrate`. The skill walks you through reclassifying each old memory as feedback or fact, rewriting the situation to a task-shaped form, dropping obsolete fields, writing to the new split layout, and archiving the originals to a dated directory. It's guided, not automatic — the classification step needs judgment.

---

## Where it's at

Still alpha, but churning less. The arc from the March post to now is two features I built and abandoned, a schema split I'm confident in, a retrieval architecture I'm mostly confident in, and a set of skills that represent my current best guess at the right shape. The corpus is smaller and sharper than the version I wrote about last month.

Source, docs, full changelog: [github.com/toejough/engram](https://github.com/toejough/engram). [Issues](https://github.com/toejough/engram/issues) welcome — especially if you ran the old version and something feels worse.
