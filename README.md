# /grade

> ## ⚠️ Before you start
>
> Your AI agent can install this skill (copy into `~/.claude/skills/grade/`, then restart Claude Code to load it). A few things need **you** or your environment:
>
> - A **browser-screenshot capability connected** (e.g. the Claude-in-Chrome MCP) — `/grade` analyzes real screenshots, so without one it has nothing to look at.
> - A **running local dev server** to point it at.


**Visual verification for AI-built UIs — honest screenshot analysis with PASS/FAIL verdicts written to `AUDIT.html`.**

## What it does

`/grade` is a Claude Code skill that proves your UI actually works instead of taking the code's word for it. It navigates your running app, captures real screenshots, and then describes *only what is visible in the image* — never what the code is supposed to do. Each feature gets a verdict: **PASS** (you can point to the pixels that prove it works), **FAIL** (something's wrong or the expected behavior isn't visible), or **?** (genuinely can't be determined from a screenshot). The results are compiled into a single self-contained `AUDIT.html` report with embedded screenshots, per-feature notes, and pass/fail/unknown filters.

The core idea is discipline against a very specific failure mode: in long sessions, an AI agent starts rushing — it screenshots to "check a box" and writes verdicts from memory of the code rather than from the image. `/grade` counters this by grading **2 features per batch** and compacting between batches, keeping every analysis fresh and honest. The golden rule is enforced throughout: *describe what you SEE, not what you KNOW.*

Part of [vibekit](https://github.com/brianharms) — a showcase of small tools and Claude Code skills that make coding with AI better.

## Install

`/grade` is a single skill file. Drop it into your Claude Code skills directory so that `~/.claude/skills/grade/SKILL.md` exists.

```bash
# Clone the repo
git clone https://github.com/brianharms/skill-grade.git

# Copy the skill into Claude Code
mkdir -p ~/.claude/skills/grade
cp skill-grade/SKILL.md ~/.claude/skills/grade/
```

Or, if you downloaded the repo as a zip, just copy `SKILL.md` into `~/.claude/skills/grade/`.

That's it — there's no install script and no companion binaries. Everything `/grade` needs (the screenshot receiver and the in-page capture helper) is bootstrapped inline at runtime from the skill itself.

Once installed, invoke it by typing `/grade` in Claude Code.

## Usage

Build a feature, then ask Claude to prove it:

```text
/grade
```

With no argument, `/grade` verifies the features from your most recent plan. You can also scope it:

```text
/grade all          # everything changed since session start (checks git log)
/grade the new sidebar filters and the empty-state illustration
```

What happens when you run it:

1. **Setup (once):** Claude confirms your dev server is running, opens a browser tab, starts a tiny local screenshot-receiver on port `9223`, and injects an `html2canvas`-based capture helper into the page.
2. **Per feature:** it navigates to the right state, screenshots the feature, writes down *only what is visible*, and issues a PASS / FAIL / ? verdict — then saves the screenshot to disk for the report.
3. **Batching:** after every 2 features it writes `GRADE_PROGRESS.json` to the project root and compacts, so progress survives context pressure and analysis stays sharp.
4. **Report:** once all features are graded, it builds `AUDIT.html` at the project root and opens it — a dark-themed, single-file report with a summary bar, All / Pass / Fail / Unknown filters, and an expandable block per feature showing the embedded screenshot and the honest notes behind the verdict.

Example progress output during a run:

```text
Graded 4/15 — PASS.
```

## Requirements / Dependencies

- **Claude Code CLI** — `/grade` is a Claude Code skill and runs inside it.
- **A way to take browser screenshots.** `/grade` drives a browser via Claude's MCP screenshot tooling (e.g. the **Claude-in-Chrome MCP**, using the `computer` tool's `screenshot` action). Without a screenshot-capable MCP connected, the skill has nothing to analyze.
- **Node.js** — used to spin up the lightweight local screenshot receiver on port `9223`. It's invoked inline; no global package install is required.
- **Internet access at runtime** — the in-page capture helper loads `html2canvas` from a CDN. (Vendor it locally if you run air-gapped.)
- **A running dev server** for the app you're grading — `/grade` verifies a live UI, not static code.

No OS lock-in: the skill is plain Markdown plus inline Node/JS, so it works anywhere Claude Code and a browser-screenshot MCP run.

## For AI coding agents

If you're an agent working **on** this skill, here's the contract.

**Repo layout:**

```
skill-grade/
├── SKILL.md      # the entire skill — this is the contract Claude reads
├── LICENSE       # MIT
├── .gitignore
└── README.md
```

**`SKILL.md` is the whole product.** Its YAML frontmatter (`name`, `description`, `user-invocable`, `arguments`) is what registers `/grade` and controls when Claude triggers it — the `description` is load-bearing for invocation matching, so edit it deliberately. The body is the procedure Claude follows. There is no `scripts/`, `web/`, `swift/`, or `template.html`; the screenshot receiver (the inline Node HTTP server on port `9223`) and the in-page capture helpers (`window._gradeCapture` / `window._gradeSave`, backed by `html2canvas`) live as code blocks *inside* `SKILL.md` and are pasted into the environment at runtime. If you extract them into separate files, update the install instructions and the runtime bootstrap together — don't let the README and the skill drift.

**How to test changes:** copy your edited `SKILL.md` into `~/.claude/skills/grade/`, start Claude Code in a project with a running dev server, and invoke `/grade`. Verify the full loop end to end: setup runs, screenshots land in the screenshots dir, `GRADE_PROGRESS.json` is written and survives a compaction, and `AUDIT.html` is generated at project root and opens.

**Invariants — do not break these:**

- **The golden rule is the point.** Every verdict must be justified by what's visible in the screenshot, never by knowledge of the code. Any edit that lets the agent assert PASS from code reasoning defeats the skill.
- **Keep the three-verdict model** — `PASS` / `FAIL` / `?`. `?` is for genuinely un-screenshottable states (e.g. interactions the MCP can't perform), never a lazy PASS.
- **Preserve batching + compaction.** Grade 2 features per batch, persist to `GRADE_PROGRESS.json` at the project root, then compact. This is what keeps long runs honest; don't raise the batch size or drop the checkpoint.
- **`GRADE_PROGRESS.json` is the resume contract.** Keep its shape (`total`, `completed[]`, `remaining[]`, `batch`, `screenshots_dir`) stable so a post-compaction session can pick up exactly where it left off.
- **Keep `AUDIT.html` single-file and zero-dependency.** Inline CSS, embedded screenshots, dark theme, summary bar, and All / Pass / Fail / Unknown filters — it must open standalone with no server and no external assets.
- **Don't hardcode machine-specific absolute paths.** Use the runtime values (`screenshots_dir`, project root) and `~`-relative paths so the skill is portable.

## License

MIT © 2026 Brian Harms / Ritual Industries — [ritual.industries](https://ritual.industries)
