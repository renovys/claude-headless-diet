# claude-headless-diet

Field notes on cutting token usage in **headless** Claude Code runs (`claude -p`) — the kind that live in cron jobs, daemons and pipelines, where nobody is watching and every run pays the same input cost forever.

Most of what follows contradicts a plausible-sounding assumption. In our automations the two largest wins were not prompt rewriting: they were **where the CLI reads its instruction files from** and **which flag actually controls the tool schema**. Everything here is written as *belief → what we measured → what to do*, and every number is one we recorded ourselves.

## How we measured

- Claude Code CLI, headless invocations (`claude -p`), models Opus and Sonnet depending on the pipeline.
- Token counts come from `--output-format json`, summing the reported `usage` fields for a run. Where a comparison is quoted, both arms used the same prompt and the same task.
- Measurements were taken in mid-to-late July 2026 on a single Linux host running ~10 headless pipelines.
- Sample sizes are small (single-digit runs per arm, unless noted). Treat the direction as the finding and the exact number as an artifact of our setup.

One measurement discipline is worth stating up front, because it is what saved us from publishing a false claim: **a flag that is documented to exclude something is not evidence that it excluded anything.** Before this work, our pipelines were annotated as "instruction files already excluded via flag". They were not. Measuring showed the flag had no effect at all, and roughly 120K tokens/day we believed we had saved were never saved.

---

## 1. `--allowedTools` does not shrink your input

**The belief.** `--allowedTools` restricts which tools the run may use, so restricting it should shrink the tool definitions loaded into the prompt.

**What we measured** (2026-07-29, same prompt, total input tokens):

| Tool configuration | Total input |
|---|---|
| default (all tools) | 56.6K |
| `--tools` with 9 tools | 41.3K |
| `--tools` with 3 tools | 31.3K |
| `--tools none` | 28.0K |

`--allowedTools` gates *permission*. The tool schemas are loaded regardless. Only `--tools` changes what is sent.

**Do this.** Decide the minimum tool set your automation actually needs and pass it to **`--tools`**. If you also want the permission gate, pass the same set to `--allowedTools` as well — they are complementary, not alternatives. For pipelines that only need the model to think and return text, `--tools none` is the floor.

## 2. `--tools` is variadic, so a trailing prompt gets eaten

**The belief.** Flags and the positional prompt can be written in any order, like most CLIs.

**What we measured.** `--tools` and `--mcp-config` take a variable number of values. Put the prompt after them and it is consumed as another value instead of being treated as the prompt. The run does not error loudly; it just does the wrong thing.

**Do this.** Put the prompt **immediately after `-p`**, or feed it on **stdin**. Never end the command line with the prompt when a variadic flag precedes it.

```bash
# good
claude -p "$PROMPT" --tools Read,Glob --output-format json
# also good
printf '%s' "$PROMPT" | claude -p --tools Read,Glob --output-format json
# bad — the trailing prompt is swallowed as another --tools value
claude -p --tools Read,Glob "$PROMPT"
```

## 3. Project instruction files load from `$HOME`, and no flag stops them

**The belief.** A run started in a clean working directory, or started with the "exclude instructions" flag, will not load your `CLAUDE.md` chain.

**What we measured.** Neither works. Our instruction chain (about 38KB across two included files) was loaded into the system prompt on **every** headless call.

| Arm | Total tokens for one run |
|---|---|
| default | 37,827 |
| exclude-instructions flag | 37,605 (no meaningful change) |
| changing `cwd` only | no change |
| isolated `HOME` | 8,849 |

That is a **77% reduction** on that pipeline, and it is the single biggest lever we found. The instruction chain resolves relative to `HOME`, not to the working directory, so only replacing `HOME` removes it.

**Do this.** Give each headless pipeline its own throwaway home directory and run with `env HOME=<apphome>` and `cwd=<apphome>`. Two caveats we hit:

- **Keep your existing auth.** Symlink only the credentials file from the real home into the isolated one. Do not let the isolated home fall back to an API key — that silently switches the run from your subscription to metered billing. We made the launcher **abort** if the credential symlink is missing, rather than proceed.
- **Strip billing-override environment variables** (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, `ANTHROPIC_BASE_URL`, and the Bedrock/Vertex switches) from the child environment for the same reason: if present, they take precedence over the subscription.
- There is also a documented "skip instruction files" flag that bypasses the chain but does not read the OAuth credentials, so it forces API-key billing. We could not use it.

### The limit: isolated `HOME` breaks file-editing tools

Read-only work survives isolation. **Writing does not.** In our runs `Read`, `Glob`, read-only `Bash` and the web tools all worked from an isolated home, but `Edit` failed there while succeeding from the real home with identical flags. We tried a minimal `settings.json` granting `Edit`, an accept-edits permission mode, and pre-accepting the directory trust flag — none of the three fixed it. The cause is state attached to the home directory, and we did not isolate it further.

Practical consequences:

- If a pipeline **reads** files outside the isolated home, add `--add-dir <path>`. We confirmed this restores read access *and* does not pull the instruction chain back in (the run stayed at 8,960 tokens).
- If a pipeline **writes** files, do not isolate it. We reverted one such pipeline after seeing edits fail silently, and we accept the cost: about 18,500 extra tokens per run on that one job.
- The structural fix, if you care enough: have the model decide and a plain script perform the write.

**Checklist before isolating a pipeline:** ① does it write files? → don't isolate ② read-only? → isolate plus `--add-dir` ③ after the change, exercise the tools for real. Passing a syntax check proves nothing here, because the failure mode is a tool that quietly stops working.

## 4. Headless runs cannot see your environment variables

**The belief.** Exporting `MODE=1` before the call is enough for the model to branch on it.

**What we measured** (2026-07-21). Only the prompt text enters the context. Given "if `X` is 1 use mode A, otherwise mode B" with `X=1` actually exported, the run's **first output was mode B — wrong** — and it only corrected itself after choosing, on its own initiative, to shell out and check. With the same instruction but the value **injected as text at the top of the prompt** (and the environment variable deliberately set to the opposite value), it answered correctly and deterministically.

**Do this.** Have the launcher decide the mode and prepend it to the prompt as a short header, and word the prompt to match ("the injected flag", not "the environment variable"):

```bash
MODE_HEADER=$(cat <<EOF
## Run mode (decided by the launcher and injected here — this is authoritative.
## Do not infer it or look it up.)
- FLAG_A=${FLAG_A:-0}
EOF
)
claude -p "$MODE_HEADER

$(cat PROMPT.md)" --tools none
```

**Why this matters more than it looks.** The failure is silent. A pipeline that publishes something every day keeps publishing; an entire branch can be dead for weeks with nothing to notice. Three of our launcher flags carried this bug at once, and all three had never fired.

### A related headless failure: "run unattended" is not a sufficient instruction

Also measured (2026-07-25, two runs): with `--tools none`, the model still spent its turn emitting *"first let me check the instructions and memory"* and then ended — producing an empty result, even from an isolated home with no tools available. The caller treated the empty string as a failure and fell through to its fallback, so again nothing looked broken.

Four things had to be pinned at the very top of the prompt before it behaved:

1. no tools;
2. no reading files, memory or instructions — **and no saying that you will**;
3. no questions, no plans, no narration of process;
4. the first output *is* the final deliverable.

And keep the caller defensive: retry once on an empty return, then fall back to a non-LLM path. An LLM failure should never take the whole job down.

## 5. Delegation only saves tokens if you pin the return format

**The belief.** Handing a big task to a sub-agent or another model keeps the work out of the main context, so it saves tokens by construction.

**What we measured.** It does not, if the sub-agent is free to answer at length. Full source files and diffs come back into the parent context and cancel the win. With the return format constrained — *final answer ≤ 20 lines, no code, no diffs, no file contents, one line of change plus one line of verification per task* — even large implementation jobs cost the parent about twenty lines. The parent then reviews with a local diff instead of by reading the sub-agent's prose.

**Do this.** Write the return contract into the delegation prompt itself, not into your own hopes.

## 6. Cap the output of every long-output command

**The belief.** You will notice when a command dumps something enormous.

**What we measured.** You notice afterwards, and by then it is in the context and in the cache for every subsequent turn — the cost is *cumulative context × number of turns*, not a one-off. One of our reference documents measured about 56K tokens to read in full; that is the price of a single careless `cat`.

**Do this.**

- Always pass a limit: `-n` / `--tail` / `--since` on log readers, `git log -n`, `head` on anything unbounded.
- For large reference documents, read the range you need (`grep -n` to locate, then a line range) instead of the whole file. Read the full file only right before replacing it wholesale.
- Do not re-read a range you have already read in the same session.
- Never dump a sub-agent transcript. If you must debug one, narrow it with `grep`/`jq`, exclude prompt and credential fields, and still cap it with `head`.
- Start narrow and widen only when the evidence is visibly truncated.

## 7. Know where your tokens actually are before optimizing output

**The belief.** Making the assistant's prose terser is a meaningful saving.

**What we measured.** Aggregating 256 session logs over seven days: **6,470M total tokens**, of which **output was 26.8M (0.41%)**, cache reads 6,274M (96.98%), cache creation 168.7M (2.61%), and uncached input 0.12M. Weighting by price (output ×5, cache read ×0.1, cache creation ×1.25 relative to input) puts output at roughly **14% of cost**. Splitting the assistant's output further: **18.6% is prose, 81.4% is tool-call arguments**.

So a tool that compresses assistant prose has a ceiling of about 14% × 18.6% ≈ **2.6%**, and an independent measurement of one such tool showed 8.5% fewer output tokens when forced on — landing at **0.2–0.7% of weighted cost**. We did not adopt it. Advertised figures of 60–90% came from chat-shaped examples, not from tool-heavy agent sessions.

**Do this.** Aggregate your own session logs before installing anything that promises savings. Optimize the fat part: instruction/system prompt loading (§3), tool schemas (§1), and the size of what you paste into context (§6).

### Corollary: don't trust a savings tool's self-reported numbers

One rewriting proxy we evaluated reported "90.8% saved". Recomputing from its own local database: **47% of the total came from a single oversized command, 66% from the top two**, and the per-command **median saving was 4.7%**. Its estimate was bytes÷4 with no tokenizer, and it counted as "saved" output that the CLI would have truncated anyway; one line item implied more input than the whole session had. We ran our own A/B instead — separate account, price-weighted metric, pre-registered decision rule — and one thing dominated everything else:

**Cache temperature is the biggest confounder in any A/B of this kind.** Two runs fifteen seconds apart scored 32,278 vs 6,408 on our weighted metric. That was not the treatment. The arm that ran first paid the cache-creation cost and the second read it back almost free. We ended up enforcing a minimum spacing longer than the cache TTL between trials, alternating arm order, holding permissions identical across arms, and discarding the trials taken before we understood this.

---

## Quick checklist

- [ ] Pass `--tools` (not only `--allowedTools`) with the minimum set; `--tools none` if no tools are needed.
- [ ] Prompt goes right after `-p` or on stdin, never trailing a variadic flag.
- [ ] Read-only pipelines: isolated `HOME` + symlinked credentials + `--add-dir`; abort if credentials are missing.
- [ ] Writing pipelines: do not isolate `HOME`; budget the extra cost or move the write out of the model.
- [ ] Strip API-key / alternate-endpoint environment variables from the child environment.
- [ ] Inject run mode as prompt text; never rely on environment variables.
- [ ] Pin the return format of anything you delegate.
- [ ] Cap every long-output command.
- [ ] Measure with `--output-format json` before and after. Twice, spaced beyond the cache TTL.

## Disclaimer

These are unofficial observations from one setup, not documented behaviour. Flag semantics, defaults and prompt composition change between CLI versions, and several of the effects above are exactly the kind that a release can silently fix or reverse. Re-measure on your own version before relying on any of it.

## License

MIT — see [LICENSE](LICENSE).
