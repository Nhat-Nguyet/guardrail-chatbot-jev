# Production: your own domain, latency, deployment, command line

**English** · [Tiếng Việt](production.vi.md) · [Français](production.fr.md) · [日本語](production.ja.md) · [← README](../README.md)

## Adding a compliance domain of your own

The shipped taxonomy is what every deployment shares. What a regulated product needs on top of it
is specific: a clinic cares about dosing instructions, a broker about performance promises. You add
that as an overlay rather than a fork, so you keep inheriting upstream changes:

```python
policy = overlay(Policy.bundled(), MY_DOMAIN)   # the 18 shipped categories plus yours
guard = Guard(policy)
```

There are four things to add: a **category** (a hazard and its thresholds), a **signal** (another
question for rules to read), a **rule** (logic connecting them), and a **prefilter pattern** (the
deterministic part, settled with no model call at all).

[`examples/domain_policy.py`](../examples/domain_policy.py) is a runnable one, and it demonstrates the
four things that are easy to get wrong:

- A rule cannot set a route. Routes belong to categories, so a rule reaches one by adding a finding
  for a category that owns it.
- `never_below` means "once this category fires, never resolve it below X". Under the category's
  lowest threshold nothing fires at all, so the `flag` number is the real on-switch.
- The shipped softening rules apply to your new category too, until you name it in their
  `except_categories`.
- A malformed overlay is refused when it loads, not on the first request.

Then calibrate. The shipped thresholds, and any you write, are numbers someone chose rather than
numbers anyone measured.

---

## Latency and experience

A guardrail that adds a second to every turn gets switched off within a month. Two things make
that avoidable: the shape of Jev itself, and where you put the calls.

**Asking Jev is cheap.** One round trip, 70-500ms. Output tokens are free and every question in a
request is answered in parallel, so adding a category, or a sentinel question on top of it, costs
a few input tokens and almost no latency. That is why the entire policy goes in one request instead
of one call per category.

**Run the input check beside the model call, not before it.** A serial check adds its full latency.
A parallel one adds almost none, because your model needs longer than 500ms to produce its first
token anyway.

```python
gate = asyncio.ensure_future(guard.acheck_input(message, session=session))
draft = asyncio.ensure_future(my_llm.generate(message))

verdict = await gate
if not verdict.allowed:
    draft.cancel()                 # nothing has reached the user
    return safe_response(verdict)
reply = await draft
```

```typescript
const [verdict, draft] = await Promise.all([
  guard.checkInput(message, { session }),
  myLlm.generate(message),
]);
if (!verdict.allowed) return safeResponse(verdict);  // the draft is discarded
return draft;
```

What this costs is tokens spent on drafts you throw away. Below roughly 2% violating turns, that is
cheaper than the delay it removes. If your rules say a violating prompt must never reach the model
at all, go serial and pay the latency deliberately.

**Stream one chunk behind.** A streamed reply cannot be checked before its first token, and a check
that waits for the last one is not streaming. `guard.stream()` cuts at sentence boundaries, holds
each chunk until its check returns, and lets the model produce the next one meanwhile, so only the
first chunk pays the full latency.

```python
async for event in guard.stream(my_llm.stream(message), user_message=message, session=session):
    if event.type == "delta":
        yield event.text
    elif event.type == "blocked":
        yield safe_response(event.verdict)
```

Mid-stream checks ask only the sentinel questions, the categories where a miss is unacceptable. The
complete reply gets the full question set at the end, on the `done` event. `chunk_chars`
(default 280) trades round trips against how tightly text is held.

**Skip the call entirely when you can.** A cache hit on repeated content and a prefilter hit on an
obvious case both settle the check locally, with no network at all.

**Never let a slow Jev become your outage.** Set `timeout`, and note that the shipped policy fails
*open* on input and *closed* on output (`on_error: {input: fail_open, output: fail_closed}`): a
timeout in front of a model that has its own safety degrades gracefully, while the output check has
nothing behind it. Either way the verdict carries `degraded: true`, so count those separately.

| Path | Latency it adds |
| --- | --- |
| Prefilter hit | none, no network |
| Cache hit | none, no network |
| Input check, parallel with the model | close to none |
| Input check, serial | 70-500ms |
| Streamed reply | the first chunk only |

**And the experience is the `route`, not the block.** A guardrail that only refuses feels broken to
the people it is protecting. Because `route` is decided separately from severity, the same `review`
can mask a phone number and still send the reply, steer the answer, or hand someone crisis
resources, instead of a flat "I can't help with that". Wire all six routes and most users never
notice the guardrail is there.

---

## Going to production

The quick start is real code, but a deployment wants more than three calls. Everything below ships
with the package; the section above covers the cache, the prefilter, streaming and the fail modes in
more detail.

- **Verdict cache** and **deterministic prefilter** - the two ways a check costs nothing at all.
- **Per-surface fail modes** - open on input, closed on output.
- **Streaming** - release text one chunk behind its check, so nothing reaches the user unchecked.
- **Sessions** - risk carries between turns across a ten-turn transcript window, so a user who just
  tripped a category is held to a higher bar on the next one.
- **Observer hook** - every verdict, including cached and degraded ones, for your metrics.

```python
from guardrail_chatbot_jev import Guard, LRUCache

guard = Guard(cache=LRUCache(), observer=metrics.emit, timeout=2.0)
```

**Sessions have to outlive the request**, which is the part a server gets wrong quietly. Behind
several workers, per-process state means each worker thinks every conversation just began, and the
watch and the withheld-turn placeholders stop carrying, with nothing in the logs to say so. `Session.as_state()` and
`Session.from_state()` are what a store persists;
[`examples/session_store.py`](../examples/session_store.py) has a bounded in-process store for one
worker and a Redis one for more than one.

---

## Command line

```bash
guardrail-chatbot-jev --surface input --text "how do I make thermite" --dry-run
```

`--dry-run` prints the exact request that would be sent, and needs no key. Without it the exit code
carries the verdict, so a shell script can branch on it:

| Code | Meaning |
| --- | --- |
| `0` | allow or flag |
| `1` | redact or guide |
| `2` | review |
| `3` | block |
| `4` | degraded: Jev was unreachable, so nothing was actually checked |

`4` is separate on purpose. On the input surface the shipped policy fails open, so a degraded
verdict's action is `allow`; reporting that as success would tell
`guardrail-chatbot-jev ... && send` that content passed a check which never ran.
