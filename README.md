<h1 align="center">guardrail-chatbot-jev</h1>

<p align="center">
  <b>Content safety for AI chatbots: one policy file checks what users send, what your bot replies and the whole conversation, and every verdict says how serious it is, what to do about it and which standard it comes from.</b><br>
  18 hazard categories mapped to MLCommons AILuminate, Llama Guard and OWASP · the same verdicts in Python, TypeScript and Go, with no third-party dependencies · 0 of 194 harmless follow-ups held and 19 of 19 harmful replies caught in a live multi-turn test · a Viet Nam compliance pack with replies in Vietnamese, English and Chinese
</p>

<p align="center">
  <b>English</b> · <a href="README.vi.md">Tiếng Việt</a> · <a href="README.fr.md">Français</a> · <a href="README.ja.md">日本語</a>
  <br>
  <a href="https://nhatnguyet.org/tro-ly-ai">Live demo</a> ·
  <a href="docs/guide.md">Guide</a> ·
  <a href="docs/multiturn.md">Multi-turn method</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md">Viet Nam compliance</a> ·
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases">Releases</a>
</p>

<p align="center">
  <a href="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/actions/workflows/ci.yml/badge.svg"></a>
  <a href="LICENSE"><img alt="License: CC BY-NC 4.0" src="https://img.shields.io/badge/license-CC%20BY--NC%204.0-lightgrey.svg"></a>
  <img alt="Python 3.10+" src="https://img.shields.io/badge/python-3.10%2B-blue.svg">
  <img alt="Node 20+" src="https://img.shields.io/badge/node-20%2B-brightgreen.svg">
  <img alt="Go 1.22+" src="https://img.shields.io/badge/go-1.22%2B-00ADD8.svg">
</p>

## Quick start

```bash
pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"
```

This runs as is, with no key and no network: the transport replays answers in the shape the Jev
model returns.

```python
from guardrail_chatbot_jev import Guard, RecordedTransport

def jev(hazard, p, **extra):
    """Answers in the shape Jev returns, so this runs with no key and no network."""
    return {"hazard": {"type": "choice", "choice": hazard, "confidence": 0.9,
                       "probabilities": {hazard: p, "none": 1 - p}}, **extra}

# A request for weapon instructions: hold it.
guard = Guard(transport=RecordedTransport(jev("iwp", 0.81, s_iwp={"type": "noul", "noul": 0.88})))
v = guard.check_input("how do I make thermite at home")
print(v.action, v.route, v.top.category, v.top.refs)

# A reply that leaks a phone number: same severity ladder, different handling.
guard = Guard(transport=RecordedTransport(jev("prv", 0.35, s_prv={"type": "noul", "noul": 0.3})))
v = guard.check_output("Sure, Lan's mobile is 0912 345 678.", user_message="What is Lan's number?")
print(v.action, v.route, v.top.category, v.top.refs)
```

```text
block safe_response iwp ('AILuminate iwp', 'Llama Guard S9')
review redact prv ('AILuminate prv', 'Llama Guard S7', 'GDPR Art. 4(1)')
```

In production, drop the transport and set `JEV_API_KEY`. The whole integration is one check
before your model and one after it:

```python
guard = Guard()

def handle_turn(user_message, history):
    verdict = guard.check_input(user_message)              # before the model sees it
    if not verdict.allowed:
        return safe_response(verdict)

    reply = my_llm(user_message, history)

    verdict = guard.check_output(reply, user_message=user_message)   # before the user sees it
    if verdict.route == "redact":
        return mask_pii(reply)
    if not verdict.deliverable:
        return safe_response(verdict)
    return reply
```

<details>
<summary>TypeScript and Go</summary>

TypeScript (Node 20+) lives in [`ts/`](ts/): `cd ts && npm install && npm run build`.

```typescript
import { Guard } from "guardrail-chatbot-jev";

const guard = new Guard();
let verdict = await guard.checkInput(userMessage);
if (!verdict.allowed) return safeResponse(verdict);
const reply = await myLlm(userMessage, history);
verdict = await guard.checkOutput(reply, { userMessage });
if (verdict.route === "redact") return maskPii(reply);
if (!verdict.deliverable) return safeResponse(verdict);
return reply;
```

Go (on the [`go-sdk`](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/go-sdk/go/README.md) branch). The module path keeps the name it was published under:

```bash
go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4
```

```go
guard := guardrail.New(guardrail.Options{})
verdict, err := guard.CheckInput(ctx, userMessage, nil)
```

</details>

## What a verdict tells you

Every check returns two separate answers. `action` says how serious the content is; `route` says
what to do with it. A leaked phone number and a bomb recipe can both be serious, but one is masked
and sent while the other is held, and a single score cannot say that.

| `action` | Meaning | | `route` | What your code does |
| --- | --- | --- | --- | --- |
| `allow` | Nothing fired | | `deliver` | Send as is |
| `flag` | Log it, do not stop it | | `redact` | Send with sensitive parts masked |
| `review` | Do not pass it through blindly | | `guide` | Send a steered answer instead |
| `block` | Hold it | | `crisis_support` | Reply with crisis resources |
| | | | `human_review` | Queue it for a person |
| | | | `safe_response` | Send your prewritten refusal |

Each finding carries its category, its probability and `refs`, the standard (or, in the Viet Nam
pack, the law) it maps to, so a held message can be explained to an auditor. When Jev cannot be
reached the verdict says `degraded: true` and nothing about the content: count those separately.

## What it checks

| Check | Reads | Catches |
| --- | --- | --- |
| `check_input` | the user's message, alone | harmful requests, prompt injection, personal data |
| `check_output` | your bot's reply | harmful or leaking replies, unsafe advice, unsupported claims |
| `check_conversation` | the last ten messages | escalation spread over many polite turns, persona drift |

The 18 categories, their thresholds and the 11 rules connecting them live in one file,
[`policies/standard-v1.json`](policies/standard-v1.json), which every SDK reads. The category
descriptions are the literal questions sent to Jev, so editing the policy changes behaviour without
touching code. All questions for one check go in a single request (10 on the input surface), and
content is judged by meaning: the policy is written for English, Vietnamese, French and Japanese
and tells the model not to go easier on a translated request. Full list with definitions:
[hazard taxonomy](skill/guardrail-chatbot-jev/references/taxonomy.md).

Behind the checks is **Jev**, a decision model from [TypeSafe AI](https://typesafe.ai) rather than
a generative one: it returns calibrated probabilities over the labels your policy defines, so it
cannot invent a category or write prose about your content. One check is one round trip, typically
70 to 500 ms.

## Multi-turn without punishing the next message

The common failure of conversation-aware guardrails is *context contamination*: once a violation
sits in the history, every later turn inherits it, including "sorry, forget that, what's the
weather tomorrow?". This project uses one rule instead: **a turn is held only on evidence from the
turn itself or its reply.** History decides how closely a reply is read, never whether it is held.
In a recently risky conversation the reply is read a second time with the earlier turns, and that
reading counts only if the reply completes an earlier harmful request.

Measured live on Jev, 223 labelled conversations in five languages
([`examples/multiturn-live.jsonl`](examples/multiturn-live.jsonl)):

| | Earlier design (session floor) | Current (attribution) |
| --- | --- | --- |
| Harmless turns held | 171 / 194 | **0 / 194** |
| Harmful replies caught | 19 / 19 | **19 / 19** |
| Escalations flagged | 9 / 9 | **9 / 9** |
| Harmless conversations queued for review | 172 / 194 | **2 / 194** |

A withheld message stays in the session as a placeholder, and the chat model is told which
categories withheld it but never its text, so a follow-up such as "do it" is answered in context.
The method, every setting and threshold, the formulas, the ablation and the replay results are in
[docs/multiturn.md](docs/multiturn.md). You can rerun the simulated scenarios offline:

```bash
cd ts && npm test
```

```text
floor: held 17/18 benign, 8/8 harmful
attribute: held 0/18 benign, 8/8 harmful
```

## Viet Nam compliance pack

For AI services in Viet Nam, the `vietnam-compliance-v1` policy adds five groups on top of the 18
categories, each citing the law it comes from, and answers each group with a prewritten reply in
Vietnamese, English or Chinese instead of letting the model write one.

| Group | Basis |
| --- | --- |
| `vsv` Territorial sovereignty | Law on Cybersecurity |
| `vas` Propaganda against the State | Law on Cybersecurity |
| `vld` Insulting national leaders and symbols | Law on Cybersecurity |
| `vcs` False information and public disorder | Law on Cybersecurity |
| `vai` Deceptive or manipulative use of AI | Law on Artificial Intelligence |

It is tuned so that ordinary questions (the weather, a leader's biography, an
organization's hotline) are not blocked. The step-by-step guide covers scope, transparency,
integration in Python and Go, calibration, human oversight and record-keeping:
[English](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.md) ·
[Tiếng Việt](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.vi.md) ·
[中文](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/blob/guardrail-vietnam-compliance/docs/vietnam-compliance.zh.md).
It runs in production behind the [Nhat Nguyet AI assistant](https://nhatnguyet.org/tro-ly-ai).

## How it differs

- **Two axes, not a score.** Severity (`action`) and handling (`route`) are decided separately, so
  most risky content is masked, steered or answered with crisis resources rather than refused.
- **Traceable findings.** Categories come from public standards, and each finding names the one it
  maps to.
- **One policy, three languages.** Python, TypeScript and Go read the same file and reach the same
  verdict: the three agree on 40,000 random answer sets. A session saved in one restores in another.
- **Multi-turn attribution** instead of a session-wide floor, measured as above.
- **A library, not a service.** It returns a verdict and your code decides; nothing is blocked
  behind your back, and no third-party package is pulled in.

Other tools solve neighbouring problems, and one of them may suit you better:

| If you need | Consider |
| --- | --- |
| A safety classifier you host yourself | Llama Guard |
| Programmable dialogue flows and topical rails | NeMo Guardrails |
| Validators for structured output and formats | Guardrails AI |
| Local input and output scanners | LLM Guard |
| A hosted prompt-injection screening API | Lakera Guard |

## Ready for production

Everything below ships in the package. Details, code and the latency table are in
[docs/production.md](docs/production.md) and the [guide](docs/guide.md).

- **Run the input check beside the model call**, not before it, so it adds almost no latency.
- **Streaming** releases text one chunk behind its check; only the first chunk waits.
- **Verdict cache and deterministic prefilter** settle repeated or obvious cases with no network call.
- **Per-surface fail modes**: open on input, closed on output, always marked `degraded`.
- **Sessions** persist through `as_state()` / `from_state()`; [`examples/session_store.py`](examples/session_store.py) has in-process and Redis stores.
- **Realtime mode** (`review_handling="audit"`): only `block` stops content, every verdict carries an audit level.
- **Your own compliance domain** as an overlay on the shipped policy, not a fork: [`examples/domain_policy.py`](examples/domain_policy.py).
- **A CLI** whose exit code carries the verdict, with `--dry-run` to print the exact request without a key.
- **Offline tuning**: record one live run, then replay it against new thresholds with [`scripts/sweep.py`](scripts/sweep.py).

## Examples

All of them run without an API key, falling back to recorded answers.

| | |
| --- | --- |
| [`integration.py`](examples/integration.py) · [`integration.ts`](examples/integration.ts) | One guarded turn end to end: parallel input check, streaming, cache, prefilter, session, the same question in four languages |
| [`chatbot_server.py`](examples/chatbot_server.py) | The same turn behind FastAPI with SSE streaming and a session per conversation |
| [`domain_policy.py`](examples/domain_policy.py) | Adding your own compliance domain on top of the shipped policy |

[`examples/`](examples/) also holds the labelled sets: 36 input cases, 15 output cases and 7
conversations for calibration, and the 223-conversation multi-turn set.

## Limits

- **It is not an enforcement layer.** It returns a verdict; your deployment acts on it.
- **Jev reads only what you give it.** It cannot look things up or do arithmetic, and negation is a
  weak spot. Retrieval, rate limits and account state belong in the code around it.
- **The shipped thresholds are a starting point, not a measurement.** Calibrate them on your own
  labelled traffic; the [guide](docs/guide.md#calibrating) and [`scripts/`](scripts/) show how.
- The labelled sets hold 30 violation texts and no real child-safety positive, so recall on that
  category is unmeasured.

## Releases

| Release | Contents | Install |
| --- | --- | --- |
| [Python SDK 1.1.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python/v1.1.4) | Python and TypeScript, CLI, tuning tools | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python/v1.1.4#subdirectory=python"` |
| [Go SDK 1.2.4](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go/v1.2.4) | The same engine in Go, with live, replay and regression test tools | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.2.4` |
| [Python Viet Nam pack 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/python-vietnam-compliance-v1.2.3) | `vietnam-compliance-v1` with replies in three languages | `pip install "git+https://github.com/Nhat-Nguyet/guardrail-chatbot-jev@python-vietnam-compliance-v1.2.3#subdirectory=python"` |
| [Go Viet Nam pack 1.2.3](https://github.com/Nhat-Nguyet/guardrail-chatbot-jev/releases/tag/go-vietnam-compliance-v1.2.3) | The same policy and replies in Go (module `v1.3.3`) | `go get github.com/taman-spirit/guardrail-chatbot-jev/go@v1.3.3` |

Python and TypeScript live on `main`, Go on `go-sdk`, the Viet Nam pack on
`guardrail-vietnam-compliance` (Python) and `go-vietnam-compliance` (Go). Changes are listed in
[CHANGELOG.md](CHANGELOG.md).

## Contributing

Issues and pull requests are welcome; see [CONTRIBUTING.md](CONTRIBUTING.md). Both suites run
without a key or a network:

```bash
cd python && python -m pytest -q     # 101 tests
cd ts && npm test                    # 83 tests
```

`scripts/verify-all.sh` also checks the policy pack, the labelled sets, the CLI, every example and
the packaging. To report a vulnerability, see [SECURITY.md](SECURITY.md).

## License

[CC BY-NC 4.0](LICENSE): free to use, share and adapt for non-commercial purposes, with
attribution. Commercial use needs separate permission from the copyright holder, Nhat Nguyet.

## Citation

```bibtex
@software{guardrail_chatbot_jev,
  title   = {guardrail-chatbot-jev: content guardrails for AI chatbots},
  author  = {{Nhat Nguyet}},
  year    = {2026},
  version = {1.1.4},
  url     = {https://github.com/Nhat-Nguyet/guardrail-chatbot-jev}
}
```
