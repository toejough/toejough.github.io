May 2026

If you're curious, v1's post is here, v2's post is here. 

V3 has arrived! What this version ships with:
* Link traversal: 
	* v2 was flat lists of memories. 
	* v3 has top level indices, or "maps of content", and drills down from there into related memories
* Harness-agnostic: 
	* v2 was pretty tightly tied to Claude Code - it looked at claude.md, claude's auto-memory space, and used anthropic API calls to haiku to make internal judgements about what memories were worth digging into, and used claude's hook/plugin infrastructure.
	* v3 uses only the binary and skills. The binary installs itself to the canonical location on your system, and installs the skills the the right places for your harness (currently opencode and claude are supported, but no reason it can't expand easily). The binary no longer makes any agent calls - all the reasoning is left to the calling agent.

This update also moves from toml format to markdown. The primary motivator here was using Obsidian vaults to store data and links.

Why Obsidian? Well, it was actually a coworker's PR against v1 that set me down this course - they wanted to visualize the memories. Obsidian already has a great link visualizer, as well as a well-understood mechanism for linking.

Beyond that, there's a lot of literature already about how humans manage their memory with Obsidian - I thought I'd tap into that and see what we can apply to the LLM space.

At the moment, v3 uses an adapted Zettelkasten framework:
* Luhmann ID's
* Permanent Notes
* Maps of Content (MOC's)

I dropped the "Fleeting" notes concept that many use, because the effort of having the LLM organize was high enough even for those that it ended up with 95% of what was needed for the "Permanent" notes.

The skills have been reduced from 4 to two: just "learn" and "recall". "Learn" now reads from the project's transcripts via the engram binary, tracking a timestamped high water mark, and only reading the transcript from that mark, updating when it's done. "Recall" now reads in the MOC's and the most connected node for each independent group of linked memories. The binary handles surfacing those, the LLM decides which seem at all relevant. Then the LLM asks the binary for links to/from those. The binary returns the new memories, and the LLM applies its judgement on _those_. The cycle repeats till the LLM has exhausted the interesting memories or it has accumulated 100 of them, whichever comes first.

The memories are still in the format of facts vs feedback. I've found that continues to be a helpful dichotomy. 

Dropping the hooks framework from claude also means that nothing fires the learn/recall automatically anymore. The skill frontmatter has been improved to try to compensate, but often you'll have to tell the agent to "learn" or "recall" (though it does do so autonomously... just not always).

As I've made a material update every month, and every month I say "here's what didn't work and what we're doing now", you might wonder if there's another one coming. I think there is. Here's what I feel is lacking right now:
* history. engram instructs the LLM to pick facts and feedback, but it can't remember what we talked about this morning if it didn't think that was obviously a useful fact or feedback in the moment. A true memory system should enable the agent to retain knowledge of a full conversation for at least _some_ period of time beyond the first "learn" request.
* curation, generally. Not all memories are created equal, and not all will stand the test of time. As we add new memories, we should be re-curating and regrouping and reorganizing. Right now this happens, but very one-dimensionally - MOC's are evaluated at the "learning" phase the memories are introduced in. If at that moment there isn't enough of a generalizable pattern, or the LLM doesn't have enough peer memories loaded to recognize one, no MOC is created, and the memories are only lightly linked.
* automatic selection. This is an extension of the prior. Right now we heavily depend on the agent itself to store and link memories, and also to evaluate its own memories & select which ones to continue evaluating on retrieval. Not only is that expensive, it's fragile and a little self-fulfilling. The agent decides at every level what is and isn't a relevant memory. I've seen it stop looking when it thinks it knows better. The problem is, it only knows what it already knows. That critical insight might be one more memory link away, but on write, the LLM doesn't know that other memory exists to link to, and on retrieval it doesn't know there's something better one hop away. I'm considering embeddings to address both problems, as well as making lookups and writes considerably cheaper and faster.
* testing. I don't mean unit testing. I mean honest benchmark eval. does this memory system actually get us what we want? Faster, cheaper first-pass success on a variety of tasks? I don't have that testing system set up.

For now, though, what's there is working better than v2, and I feel like the rest of this is reasonable extensions beyond that. Give it a try and let me know what you think, or reach out with alternative solutions that I should be using instead - I'm not afraid to let this project go if something clearly better already exists!