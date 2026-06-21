+++
title = "CI logs are mostly noise. I built a tool that reads them for you."
date = 2026-06-20T12:00:00+05:00
draft = false
tags = ["ci", "devops", "kubernetes", "tooling", "ci-medic"]
summary = "A failed CI run dumps thousands of lines to tell you one thing went wrong. ci-medic distills the log, redacts secrets, and posts a verdict (what failed and why) where you already look. Here is how it does on 13 real failed logs."
+++

A CI pipeline fails. You click to see the logs. Somewhere in four thousand lines of
setup logs, dependency resolution, compiler output, and teardown is the one line
that actually matters, and you scroll until you find it.

You do this several times a day. Everyone on your team does. It is one of the most
repeated, least interesting tasks in software, and it scales linearly with how
much CI you run.

Wouldn't you rather have something read those four thousand lines for you, find the one that matters, and tell you whether to fix it or just retry? And do it without locking you to one LLM (Anthorpic, OpenAI etc.) or one CI provider (GitHub, GitLab, Jenkins)? That is **ci-medic**.

I build and operate a Kubernetes platform and maintain CI for a few
open-source projects, including the RISC-V build matrix for [llama.cpp]. "Read the
log, find the real error, decide if it's worth a retry" is a tax I pay constantly.
So I built a tool to pay it for me.

## What ci-medic does

When a pipeline fails, **ci-medic** reads the failed job's log, throws away the
noise, and posts a structured verdict where you already look: a sticky comment on
the pull request for GitHub Actions, the build description for Jenkins.

Every verdict contains:

- **category**: one of `code`, `flake`, `infra`, `dependency`, or `config`
- **root cause**: one paragraph in plain language
- **suggested fix**: the concrete next action
- **retry recommended**: whether it's worth re-running (i.e. is it a flake)
- **evidence**: the exact log lines it based the verdict on, secrets redacted

That last distinction, `flake` versus everything else, is the one that saves the
most time. "Should I just re-run this?" is the question you ask most, and ci-medic
answers it directly.

## How it works

The interesting part isn't the model call. It's everything before it.

A raw CI log is far too long and too noisy to hand to a model directly, and full
of secrets you don't want to send anywhere. So ci-medic runs a **deterministic
preprocessing pipeline** first:

1. **Strip** ANSI colour codes and normalise the text.
2. **Drop** timestamps and collapse repeated lines (CI logs repeat a lot).
3. **Redact** secrets, locally, before anything leaves the machine.
4. **Extract** the highest-signal windows: the regions around real error markers,
   weighted toward the end of the log where the actual failure usually lives.

Only that distilled, redacted window, typically a few hundred lines rather than
several thousand, is sent to a model for the verdict. The model never sees the raw
log.

## The complete list of benefits

This is the full case for using it:

1. **It finds the error for you, automatically.** No clicking into the run, no
   scrolling, no copy-pasting the error into a box. When a job fails, the verdict
   is already waiting where you look.

2. **It tells you whether to retry.** The `flake` category answers the single most
   common CI question ("is this real, or should I just re-run it?") directly, so
   you stop wasting reruns on real failures and stop debugging transient ones.

3. **It is cheap.** Logs are distilled to a small window before the model sees
   them, so a typical analysis costs a fraction of a cent, or nothing at all on a
   free or local model.

4. **It keeps your logs private.** A deterministic step redacts secrets locally
   before any model call. Verified coverage today: AWS access keys, GitHub tokens,
   and bearer/JWT-style tokens.

5. **It can run with zero data egress.** Point it at a local model (Ollama,
   llama.cpp's `llama-server`, or any OpenAI-compatible endpoint) and no log
   content ever leaves your network.

6. **It is model-agnostic.** Bring your own provider: OpenRouter, OpenAI,
   Anthropic, or a local server. You are not locked to one vendor's model.

7. **It is CI-agnostic.** The core just consumes a log file, so it isn't tied to
   one CI host. GitHub Actions and Jenkins work today; GitLab is next. It works on
   self-hosted CI that hosted assistants can't reach.

8. **It is open source.** Apache-2.0. Read it, fork it, vendor it, audit exactly
   what gets sent and what gets redacted.

9. **It posts where you already are.** A sticky PR comment on GitHub (updated in
   place, never spammed) and the native build description on Jenkins. No new
   dashboard to check, no extra tab.

10. **It shows its evidence.** Every verdict includes the exact log lines it
    reasoned from, so you can trust it or overrule it without re-reading the whole
    log yourself.

## Does it actually work? 13 real logs.

Here is a live pull request where ci-medic triaged five different deliberately-broken jobs and posted its verdicts as a comment: [ci-medic-playground PR #1](https://github.com/alitariq4589/ci-medic-playground/pull/1). Every verdict in this PR is something you can click through and verify, including the redacted AWS key in the fake-secret-leak job's evidence.

Synthetic examples prove nothing, so I also ran ci-medic against **13 real failed CI
logs**, from the llama.cpp build matrix, a Kubernetes RISC-V project, and a
deliberately-broken playground. I manually audited these to see if the reasoning held up. It's a small, qualitative sample size to prove the concept, not a definitive statistical benchmark.

A representative set, all verdicts the tool produced:

| Failure | ci-medic verdict | Right? |
|---|---|---|
| `assert 2 + 2 == 5` in a test | **code** (99%), wrong assertion | yes |
| `def f(:` syntax error | **code** (97%), invalid function def | yes |
| `import nonexistent_module` | **code** (97%), ModuleNotFoundError | yes |
| `exit $((RANDOM % 2))` | **flake** (97%), nondeterministic, retry | yes |
| `go.mod requires go >= 1.25, running 1.24` | **config** (94%), toolchain mismatch | yes |
| Bus error in perplexity test (core dumped) | **code** (78%), deterministic crash | yes* |
| NuGet 401 Unauthorized from Azure feed | **dependency** (96%), feed auth | yes |
| `EACCES: permission denied, rmdir` on runner | **infra** (75%), runner cleanup perms | yes |
| Python 3.11 not in runner cache | **infra** (96%), version unavailable | yes |
| HF model download HTTP 404 | **dependency** (94%), missing model artifact | yes |
| Model download truncated (ContentTooShort) | **flake** (78%), transient network, retry | yes |

On these, ci-medic classified the failure correctly and identified the real
causal line in every case. The one marked with an asterisk, the bus error, is
defensible either way (`code` versus `infra`), and the model flagged its own lower
confidence at 78%, which is the behaviour I want: less certain on genuinely
ambiguous failures, not falsely confident.

The honest caveats about these tests is **13 logs, hand-checked, not a labelled benchmark**,
so I won't quote a percentage that implies more rigour than that. These verdicts
all came from a free open-weights model, so results will vary by model. And the
sample skews toward build, test, and dependency failures because that's what my CI
actually produces.

But the pattern held: the deterministic distillation reliably surfaces the right
evidence, and given good evidence the verdict is usually right.

## Secrets never reach the model

CI logs leak credentials constantly: a key echoed into the environment, a token in
a URL. Since ci-medic's whole pitch includes sending the log to a model, it has to
redact first, and I won't claim coverage I haven't tested. Verified so far: AWS
access keys, GitHub tokens (full-length), and bearer/JWT-style tokens are replaced
with a placeholder before the model call. You can check any format yourself by cloning the project and running following command in root dir:

```bash
python3 -c "from ci_medic.preprocess.redact import redact; print(redact('YOUR_TEST_STRING'))"
```

For zero egress, point it at a local model and no log content leaves your machine
at all.

## How this differs from what's out there

There are AI CI-log tools already, so why this one? The hosted assistants (like GitHub Copilot or GitLab Duo) are platform-locked and send your logs to their backend. The self-healing AI agents (like Dagger or Harness) go further by opening fix PRs, but they are incredibly heavyweight and framework-specific. 

ci-medic is the lightweight, open-source middle: it triages on any CI (including self-hosted Jenkins), with any model (including local ones), and it redacts secrets before the model ever sees the log. It is for teams who want the triage without sending their logs to someone else's service.

Here is the breakdown of why this approach matters:

1. **It runs automatically and finds the line for you.** No clicking into the log, no pasting the error into a chat box. The verdict is waiting on the PR when you look.
2. **It works where hosted assistants don't.** It runs on self-hosted Jenkins today, GitLab next. A tool locked to one CI host cannot help teams running their own bare-metal runners.
3. **Your logs don't have to leave your network.** Redaction plus bring-your-own model (including an air-gapped local LLM) is not something a hosted SaaS button can offer.

## Try it

ci-medic is open source (Apache-2.0). GitHub Actions setup is a few lines in your
workflow; Jenkins needs the tool on your agent plus a `post { failure }` block.

- Repo and docs: **github.com/alitariq4589/ci-medic**
- It's v0.1. GitHub Actions and Jenkins work today. GitLab, a native Jenkins
  plugin, and cross-run flake memory ("this test has failed 6 times this month")
  are on the roadmap.

If you try it, I'd genuinely like to know where the verdict was wrong. Those are
the cases that make the next version better. I am open to contributions :) .

[llama.cpp]: https://github.com/ggml-org/llama.cpp