# Learning Prompt Hygiene

This is a Claude conversation after a series of qwen 3.8 context window overruns and recoveries. It is how I came to "prompt hygiene" – Saving tmp/session-restore-packet.md with a seed prompt followed by a /compact directive (in session, at around the ~200 message marker). Also a better understanding of **reasoning** , **context windows**, **sessions**, and what is stateful with AI.

## Conversation

> Correct if wrong: don't eject the model, reconfig load parameters, etc to avoid a prefill penalty. Just start a new chat session with the seed message from the latest session resume packet. The rest of the session resume packet is just a documentation trail for me.

Correct on both counts.

**Why no reload/reconfig is needed:** the context ceiling is a property of the *request* (how many tokens are in the prompt you're sending), not something the loaded model instance "uses up" and needs resetting. Nothing about the previous session's size persists on LM Studio's side once you stop sending that history — the model is just sitting there ready for the next request, whatever size it is. Ejecting/reloading or changing `--context-length` would only matter if you wanted a *different* ceiling than 262144, which you don't; you just want to stop feeding it a request that's already near or past that ceiling.

**Why the seed message alone is what to send:** it's specifically written to be self-contained — goal, current state, exact code changes, and the ordered next steps are all in there. A new chat thread starts with an empty history, so pasting just the seed message means the very first prompt is small (a couple thousand tokens, not 190K+), and normal prefill applies to that small prompt — no special cost-avoidance maneuver required beyond simply not carrying the old thread forward.

**The rest of the packet** (goal/checklist framing, the "don't re-litigate" decisions, the file-by-file breakdown, environment notes, diagnostic note) is exactly what you said — a record for you to sanity-check the seed message against, or to dig into if the new session ever seems to be missing context the seed message didn't fully capture. The model never needs to see that framing directly; it's redundant with the seed message by design.

> OK so the context is filled every request, so on these very long refactoring excursions, yes I need the biggest context I can run, but yes I need to periodically /compact because the whole thing will just keep growing. Eventually I will always hit a size limit if it goes on long enough. That's why reasoning is a double edged sword.

Yes, all correct — and that last line is the crux of it.

**Every request re-sends the whole thing.** These aren't stateful sessions on the server side; each turn's prompt is system + tools + full prior history + new message, from scratch. So context size only ever moves in one direction across a session — up — until something (you, via `/compact`, or a hard error) resets it.

**Bigger ceiling buys you runway, not immunity.** 262144 vs. some smaller number just changes how many turns you get before the wall, not whether there's a wall. For an open-ended refactor with no natural stopping point, you will eventually hit it — the only question is how many turns in, and how gracefully you handle the approach versus how KO's your last attempt was.

**Reasoning being double-edged is exactly right, and it's worth being precise about why:** the reasoning is genuinely what makes a 27B-class local model useful on things like the trusted-types bug hunting you were doing — it's doing real work figuring out where the escaping breaks, tracing the shadow-DOM crash, etc. But every token of that reasoning gets counted in `completionTokens`, and depending on how faithfully the harness prunes old thinking from history (which, per what we found in your export, is inconsistent), a chunk of it compounds into next turn's `promptTokens` too. So the exact thing making the model good at the task is also the thing burning through your ceiling fastest — a single hard problem that takes 50K tokens of thinking to solve costs you as much runway as 5-10 turns of ordinary back-and-forth would.

**The practical implication for how you work these long excursions**, given all that:
- Treat `/compact` as routine maintenance, not damage control — doing it at natural checkpoints (finishing a TODO item, like this session did) costs you a small one-time summarization pass in exchange for a much longer runway afterward, versus waiting until you're near the wall and either losing work or scrambling to reconstruct it after a hard failure.
- The checkpoint file your agent wrote unprompted at the turn-9 pause is worth treating as a pattern rather than a one-off — asking it to periodically write that kind of state-of-play file (not just before you step away, but before/after each `/compact`) gives you a paper trail that's independent of whatever Copilot's summarization actually decided to keep, which matters given how unreliable the automatic summarization was in the reports we found earlier.

> On cloud services like this, how is this handled? You just pay an indefinite, growing cost?

Not fully — but you're right that the *shape* of the problem is identical, and it's worth being precise about what's actually solved versus just made less painful.

**What's the same as your local setup:** every API call still resends the full history (system + tools + conversation), so token count still grows every turn, still has to fit under a ceiling, and yes, you're billed on that growing size. Cloud providers don't have some trick that avoids resending history — that's just how these models work, local or hosted.

**What cloud specifically buys you:**

1. **Prompt caching cuts the *compute cost* of the repeated portion, not the token count.** With Claude's API, you can mark a stable prefix (system prompt, tool definitions, earlier conversation) as cacheable; the next call reusing that same prefix reads it at roughly 10% of normal input price instead of full price, per the docs. First call pays a small write premium (1.25x for a 5-minute cache), subsequent calls within the TTL get the discount. This is directly analogous to llama.cpp's KV-cache-reuse-on-shared-prefix behavior you already benefit from locally — same mechanism, cloud version just turns it into a *billing* discount rather than just a latency win.

2. **It doesn't touch the context ceiling.** The cache makes reprocessing a repeated prefix cheaper and faster; it doesn't let you exceed the model's max context. So the growth problem itself — needing to `/compact` eventually — is unavoidable in exactly the same way as your local setup. Bigger context windows (Claude's models run 200K–1M depending on which one) buy more runway before that wall, same trade-off as raising LM Studio's ceiling.

3. **Yes, cost genuinely does grow with conversation length even with caching**, because the *new* tokens each turn (your latest message + the model's response) are never cache hits — only the unchanged prefix is. A long agentic session with big outputs still accumulates real spend, caching or not. It's mitigated, not eliminated.

So the honest framing: cloud services turn "pay full compute for the same tokens every single turn" into "pay full compute once, then a steep discount on rereads" — which meaningfully softens the cost curve — but the underlying fact you identified stays true either way: an open-ended conversation eventually needs compaction, because nothing removes the requirement to keep re-sending everything that came before.