# continuity

A Claude Code plugin that keeps a long session's sense of continuity across context compaction.

When a conversation fills the context window, Claude Code compacts it: everything so far is replaced by one summary. A summary keeps *what* happened and tends to lose *why*: the reasons behind decisions, the options turned down, the open threads, the feel of the collaboration. This plugin adds three things around every compaction of the main conversation:

1. **A handoff note, written with everything still in view.** Just before the compaction, a fork of the session (the session's own model, over its own transcript, served from the prompt cache) writes a note to itself:
   - where we are;
   - decisions and why;
   - open threads;
   - how the user works;
   - the feel of it;
   - in our own words: six to ten short exchanges copied word for word, because the actual lines carry a texture that no description of them does;
   - where to look.

   Each quoted line is checked against the conversation: one found there is kept with the conversation's own characters (case, punctuation, typos), one that isn't is left out, and a closing line says how many were found. Each note is saved under `~/.claude/continuity/<session id>/`.
2. **Standing instructions for the summariser**, for automatic compactions too: keep decisions with their reasons, the options rejected, every open thread and the current state. They come after anything typed after `/compact`.
3. **The note comes back right after the summary**, as Claude's own message. It carries the path of the session's full transcript, which keeps everything from before the compaction, so details can be looked up there.

`/handoff` writes a note on demand, without compacting: a checkpoint you can read. It says how many quoted lines were found word for word.

A session that was just resumed has nothing to fork until its first turn ends. In that case, the session's own model writes the note from the transcript's text instead. If anything else fails (the model doesn't answer, an error), the compaction is Claude Code's own, as if the plugin weren't there. A subagent's own compactions pass through untouched.

## What this fork adds

This is [LuxiaSL](https://github.com/LuxiaSL)'s fork of [gabriel-berzescu/claude-continuity](https://github.com/gabriel-berzescu/claude-continuity). It adds the following, all tested with `claude plugin test .`:

- **A note even right after a resume.** Upstream writes no note when there is nothing to fork (a resumed process before its first turn ends). This fork has the session's model write the note from the transcript's text instead, and its header names the fallback.
- **A "State and identifiers" section**, and a budget of about 3,000 words. The note keeps exact ids, paths, branches, commands and numbers, which summaries tend to drop. In a test over 25 real compactions, a session kept 66% of what mattered from the summary alone and 88% with the note.
- **Marks.** A tool, such as [orchestra](https://github.com/LuxiaSL/orchestration)'s `mark()`, can append moments that must survive word for word to `<continuity dir>/marks.jsonl`. The plugin copies them into the next note under "Marked to keep". The plugin's code copies them, not the model, so the wording can't drift.
- **Quotes from messages that arrive over a channel.** Messages pushed into a session over an MCP channel are kept apart from typed turns, so quoting one used to fail the check. The plugin now also checks quotes against `<continuity dir>/channel.jsonl`, which orchestra writes. Lines that are left out are saved beside the note as `handoff-*.dropped.md`.
- **Cost records.** No transcript records the note's fork, the fallback, or the compaction's own summary request. The plugin appends each one's token usage, cache reads and writes included, to `<continuity dir>/costs.jsonl`.

## Install

At the prompt of a Claude Code terminal session:

```
/plugin install continuity --marketplace LuxiaSL/claude-continuity
```

Answer `y` to add the marketplace, then pick a scope (user, so it runs in every session). From a shell, use `claude plugin install continuity --marketplace LuxiaSL/claude-continuity`. In the desktop app, where `/plugin` does not run, click + next to the prompt box, then Plugins, Add marketplace with `LuxiaSL/claude-continuity`, then Add plugin. For the original plugin, use `gabriel-berzescu/claude-continuity` instead.

While developing, load it from a working copy instead: `claude --plugin-dir <path to this folder>`, or name the folder in `CLAUDE_CODE_PLUGIN_DIRS`.

It is a hooks module (a "mod"), built on Claude Code's early-access function-hooks API, and tested with Claude Code 2.1.310.

## Goes well with

- A **Compact instructions** section in your `CLAUDE.md`. The compactor reads it, and the heading is free-form. Say what to keep: decisions and reasons, open threads, state, the collaboration's tone.
- A larger auto-compact window (`/autocompact`), so compaction happens rarely, and `"idleCompaction": false` in your settings if you don't want a session compacted while it sits idle.

## What it costs

At each compaction, one extra model request over the transcript (mostly read from the prompt cache) and a note of about 2,000 to 2,500 words. The fallback after a resume has no cache to share, so it reads up to about 150K tokens of transcript text. `costs.jsonl` records what each request used. Checking the quotes takes a few tens of milliseconds, even over a long session. `/handoff` costs the same.

## Develop

```sh
claude plugin validate .
claude plugin test .
```

`tests/continuity.test.ts` stands in for the engine: the fork, the files, the compaction. A live check: run a short session with `claude -p --input-format stream-json --plugin-dir .`, send a few messages, then `/handoff`, `/compact` and a question that needs what was said before.

## License

MIT
