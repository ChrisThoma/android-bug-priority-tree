# android-bug-priority-tree

An agent skill for [Claude Code](https://claude.com/claude-code) and [Codex](https://developers.openai.com/codex/cli) that guides a long, emulator-driven Android bug hunt around user journeys, UI functionality, understandable state, consistency, recovery, edge cases, and demonstrated user impact.

Point a few agents at an app and tell them to "find bugs" and you get one wave of easy findings, then diminishing returns, then a lot of tokens spent re-walking the same happy paths. This skill gives an external goal/orchestration loop a testing method: map meaningful user goals, verify state transitions instead of counting visited screens, record coverage gaps, and escalate into relevant conditions only after normal journeys have real depth.

Severity runs S1 to S4: S1 is catastrophic harm, S2 is task failure or a harmful result, S3 is a material UX failure, and S4 is pure polish. Severity follows the demonstrated user consequence rather than the audit category. S1 through S3 count toward the goal; at most two S4 findings may count so agents cannot satisfy a hunt with alignment nits.

This is the Android sibling of [ios-bug-priority-tree](https://github.com/ChrisThoma/ios-bug-priority-tree). The method, track interface, ledger format, and severity ladder are the same, so one host loop can drive either.

## Install

With the skills CLI (works for Claude Code, Cursor, Codex, and others):

```bash
npx skills add ChrisThoma/android-bug-priority-tree
```

Or clone the repo and copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/ChrisThoma/android-bug-priority-tree.git
cp -r android-bug-priority-tree/skills/android-bug-priority-tree ~/.claude/skills/
```

For Codex, copy into `~/.codex/skills/` instead — both hosts read the same `SKILL.md`, only the directory differs. Or for a single project, copy into `.claude/skills/` (or `.codex/skills/`) inside the repo.

## Use

Invoke it directly with `/android-bug-priority-tree` in Claude Code or `$android-bug-priority-tree` in Codex. It is a policy and testing-method companion to a "find N distinct issues" goal loop, not an executable test harness. The host loop owns builds, turns, commits, issue status, and scheduling. Load the skill before the first finder wave so its Track A-D interface, severity ladder, app-map guidance, and escalation rules apply from the beginning. Each finder receives only its own track section plus the shared budget and app map.

## What it does

1. **Map** the app in `APPMAP.md` as user goals, entry paths, prerequisites, outcomes, meaningful state branches, test data, and coverage gaps, seeded from the manifest and navigation graphs. Existing screen-only maps are migrated before reuse. Screens remain navigation landmarks; they are not the unit of coverage.
2. **Test** through four stable tracks. Track A completes journeys and verifies data and persistence. Track B inspects visual-functional state and consistency during real journeys. Track C connects source risks and hidden routes to user impact. Track D measures named, high-value journeys when a symptom warrants it.
3. **Probe** state transitions. A normal journey check follows baseline, action, immediate feedback, settled outcome, cross-view consistency, and return or relaunch, then applies relevant validation, cancel, system Back, retry, duplicate-action, rotation, process-death, and recovery cases.
4. **Escalate** from uncovered normal states into applicable permissions, deep links, dates, locales, network conditions, window sizes, motion analysis, and disruptive tests. Repository priors specialize product facts and known risks without silently replacing journey-first ordering. Accessibility-specific auditing is a late tier unless explicitly requested as a current-hunt objective; using the UI Automator hierarchy to drive the emulator is only automation plumbing.
5. **Demand impact** before counting a finding. A checklist match is not automatically S3, ambiguous product intent is a product question, and ordinary source smells remain leads until runtime behavior confirms them.
6. **Budget** context with fresh agents, assigned device serials, bounded evidence, map deltas, and concise reports.
7. **Verify** blind. Verifiers receive the defect, repro, app map, APK path, and environment, but not source, diffs, history, commit messages, or implementation notes. They repeat the user journey and one named adjacent invariant.

See [SKILL.md](skills/android-bug-priority-tree/SKILL.md) for the full tree.

## Differences from the iOS skill

- Every command targets a device serial with `adb -s <SERIAL>` instead of a simulator UDID.
- Rotation and process death (`am kill` while backgrounded) are Tier 0 Track A probes, because activity recreation is the most common Android state-loss source.
- System Back, edge-to-edge insets, and three-button versus gesture navigation are part of the core rubric.
- Crashes and ANRs are confirmed from `logcat -b crash` and `ANR in` lines rather than DiagnosticReports.
- Offline and slow networking move from the disruptive tier to Tier 2, since emulator network controls are per-instance and do not touch the host.
- Emulator snapshots are the recommended environment reset between workers.

## Requirements

Baseline requirements are Android SDK platform-tools (`adb`), the Android Emulator with one AVD per concurrent worker, an image downscaler (`sips` on macOS or ImageMagick), and a server that can drive a specifically assigned device, such as [mobile-mcp](https://github.com/mobile-next/mobile-mcp); substitute your own server's equivalents. `ffmpeg`, a root-capable system image (for timezone and system-locale changes), and LeakCanary or a heap analyzer are conditional requirements for motion, timing, or memory escalation rather than prerequisites for every hunt.

On Codex, subagents need the `multi_agent` feature, which is stable and on by default (`codex features list | grep multi_agent`). On Claude Code, no extra setup is needed. The `agents/openai.yaml` in the skill folder supplies Codex's display name and default prompt; Claude Code ignores it.

Use disposable emulators, sandbox accounts, Play Billing test accounts, and synthetic data. Do not aim destructive or repeated-action probes at production services. Re-check commands against the installed platform-tools, emulator, and system image API level before relying on them.

## License

[MIT](LICENSE)
