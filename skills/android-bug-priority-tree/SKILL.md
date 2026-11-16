---
name: android-bug-priority-tree
description: Journey-centered finder tracks, severity, and escalation for an automated Android bug hunt. Load at the START of the hunt to discover and test user journeys for UI functionality, understandable state, consistency, recovery, process death, edge cases, and measurable user impact while preserving Track A-D orchestration.
---

# Android bug hunt: user journeys, impact, and escalation

Companion to a "find N distinct issues" Android goal protocol. Load this skill before the first finder wave. The host protocol owns turns, builds, commits, issue status, fixes, and verifier scheduling. This skill owns app-map guidance, finder methods, severity, evidence, wave composition, and per-track escalation.

The four track names are a stable interface. Keep Track A through Track D in prompts, ledgers, resume logic, and `HUNT.md` forwarding. The UI Automator view hierarchy is an emulator navigation and observation tool; using it does not make the hunt an accessibility audit. Accessibility-specific defect hunting is a late escalation unless the host prompt explicitly requests it earlier.

## Host setup

The skill is host-agnostic. Install it at `~/.claude/skills/android-bug-priority-tree/` for Claude Code or `~/.codex/skills/android-bug-priority-tree/` for Codex. Per-project skill directories work too.

Baseline requirements:

- A device-driving server exposing targeted tap, text entry, element lookup, hierarchy listing, and screenshot calls against a named device serial. The examples call these the server's "find-element" and "list-elements" calls; substitute your server's names. `adb shell uiautomator dump` is the fallback full hierarchy dump.
- Android SDK platform-tools (`adb`) and the `emulator` binary on PATH, with an AVD per concurrent worker.
- Host GPU rendering: create AVDs with `hw.gpu.enabled=yes` and `hw.gpu.mode=host`, or boot with `-gpu host`; it works with `-no-window`. Save the `baseline` snapshot under that mode: a snapshot saved under another renderer fails to load ("different renderer configured") and the emulator cold-boots. At setup, check one frame rate on the launched app. Use `dumpsys gfxinfo` for views and Compose. For a GL `SurfaceView` it reports 0 frames, so read `dumpsys SurfaceFlinger --latency '<SurfaceView ... (BLAST)#n layer>'` and count frames over the timestamp span. Under about 30 fps, fix rendering before wave 1. On 2026-11-15 a GLES app on software GL ran at about 7 fps, finders came back partial, host load reached 25, and an emulator shut itself down twice. With `-gpu host` the same AVD ran at 60 fps.
- Shared AVDs. When several apps' hunts use the same AVDs, `baseline` is the shared clean image and nobody saves over it. Each app boots from its own snapshot, `<app>-baseline`, saved once at setup with its APK installed and the environment baseline applied. Launch with the adb server port in the same command: `ANDROID_ADB_SERVER_PORT=<port> emulator -avd <AVD> -port <port> -no-window -no-audio -no-boot-anim -gpu host -snapshot <app>-baseline -no-snapshot-save`. Exported afterwards, the emulator registers with the default server and the hunt's server never lists it; on 2026-11-16 that cost a setup worker 3 minutes. Claim each serial with the machine's device-claim tool (`sim-claim emulator-<port>` on Chris's Mac) before booting it. When the hunt ends, shut down what you booted and release the claim. Per-hunt lock files cover only the workers inside one hunt: on 2026-11-16 Curiosity Jar's host had to ask other sessions who held the Phone AVDs, while Koimono's ledger read turn 40 of 40 and its release smoke test still held one.
- An image downscaler: `sips` on macOS, or ImageMagick (`magick`) elsewhere.
- Fresh subagents for finder waves and verification.

Conditional tools:

- `ffmpeg` for motion analysis and video-based timing.
- A root-capable emulator image (Google APIs or AOSP, not Google Play) for `adb root` conditions such as system timezone, system locale and the clock. Those images have no Play Store, so in-app review, Play Billing and store links are blocked entry points there; record them in the map rather than assigning them. Root also unblocks multi-touch: `sendevent` is denied for the shell user, but works after `adb -s <SERIAL> root`, so a pinch or two-finger gesture is not `blocked` on these images. Never `adb unroot` to clean up: on 2026-11-16 an unroot killed an emulator mid-hunt. Restore by command instead. On 2026-11-16 root clock and timezone probes found two widget defects after 13 dry finder runs.
- LeakCanary in the debug build, or `am dumpheap` plus a heap analyzer, for targeted memory investigation.

Every emulator-bound prompt names exactly one device serial (for example `emulator-5554`). Every `adb` call must pass `-s <SERIAL>`; never rely on the default device when more than one is attached, and never set `ANDROID_SERIAL` globally for concurrent workers. Launch concurrent instances of one AVD with `-read-only` or use distinct AVDs. Give concurrent workers unique artifact directories and filenames, both on the host and under `/sdcard/`.

## Safety and test contract

Before hunting, establish what can be inferred from the repository and host prompt:

- Supported build variant, `minSdk`/`targetSdk`, form factors, orientations, declared permissions, exported components, account roles, and integrations.
- Test account, backend, fixtures, reset method (`pm clear`, a debug wipe extra, or reinstall; a snapshot only at boot), and whether destructive actions are safe.
- Product promises from requirements, tests, visible UI copy, and established behavior elsewhere in the app.
- Known limitations, unsupported conditions, and environment constraints.

Use sandbox accounts, Play Billing test accounts or license testers, and synthetic data. Do not make real purchases, mutate production data, expose credentials or personal data in evidence, or perform irreversible external actions without explicit authorization. Disruptive manual conditions and anything that affects the host or other emulators require user approval and a restoration plan.

A finding's expected behavior must be grounded in a visible UI promise, documented requirement, data invariant, established equivalent behavior, or strong Android or Material convention (system Back, state surviving rotation and process death, edge-to-edge insets). If the expected behavior is genuinely ambiguous, report a `PRODUCT QUESTION` lead rather than a candidate. An expected basis read only from source (a constant, a comment, a code path the UI never promises) is ambiguous too: on 2026-11-14 an iOS hunt counted and fixed "say the forecast uses a 28-day default" without the user ever being asked. The reverse also holds: replacing or deleting the user's data with no confirmation and no undo (a restore, a reset, an import) breaks a strong platform convention and is a candidate, S2 when logged data is lost, not a product question. Unreachable functionality is a coverage gap, not a defect, unless its entry point should be available to the tested user.

## APPMAP.md: journey map

Create `APPMAP.md` before wave 1 from bounded source inspection, `AndroidManifest.xml`, navigation graphs, app metadata, launch behavior, and supplied product information; runtime finders then return concise deltas when they discover new paths or prerequisites. If a map already exists, validate and migrate it to the journey schema below before reuse. Do not treat a screen list as a journey map. The orchestrator is the single writer. When a setup worker drafts the first map, its prompt carries this section's journey rules word for word, above all that conditions such as rotation, process death and font scale are never JIDs. On 2026-11-16 a paraphrased map prompt came back with persistence and rotation as J10 and J11.

Keep `APPMAP.md` compact and structural. It describes the app, not the hunt: it holds no coverage state, issue numbers, verdicts, wave status, or hunt-relative wording such as "this hunt" or "this session." Coverage lives in the ledger's `## Coverage` section (below); defect status lives in the ledger's issue rows. A map may cite a prior hunt's finding only as `round-N #M` for an intentional-behavior or dead-lead note. One target (package and build variant) per map; never widen the target mid-hunt.

When migrating or trimming a map, these survive every migration and are restated, never deleted: install/launch/reset commands, fixtures and test-data sentinels, capabilities and blocked entry points, intentional-behavior notes, tooling caveats and traps, `unsupported` markers, dead leads that were investigated and disproven, and the installed source revision line. Restamp the revision line to the current HEAD after each fix install; a stale stamp is corrected, not removed.

An intentional-behavior note records a product decision and its source (UI copy, a spec, the user's word). How the code happens to behave ("no saved instance state; an open dialog resets on recreate") goes under a separate `observed implementation:` label, and it never waives a finding: a strong platform convention still sets the expected basis. On 2026-11-15 finders read such a line as intended and dropped text typed into a rename dialog and lost on process death ("dropped (documented)"), which the severity ladder rates S2. Numbers or behavior carried over from another platform's map (an iOS sibling's timings, thresholds or counts) are checked against this target's source, or marked `unverified` until they are.

Give every subagent the map atomically. In a shared workspace, prefer an explicit first action to read the exact repo-root `APPMAP.md`; otherwise paste its full contents. Never use placeholders, summaries, ellipses, or "same as above." Before dispatch, reject any prompt containing an unexpanded map/track placeholder. A worker must read the map before emulator or source work and identify its assigned JID.

For each meaningful journey, record:

```
JID | user goal | entry path | prerequisites/test data | expected outcome
states/branches: success, validation, cancel, loading, empty, error, retry, completed (applicable only)
notes: intentional behavior, traps, and intent extras or launch flags specific to this journey
```

Also record the package name, launcher activity, exact install/launch/reset commands, the per-emulator environment baseline (locale, night mode, font scale, display density, animation scales, navigation mode, power state), stable screen names, global navigation (bottom bar, drawer, system Back behavior per screen), test-data sentinels, declared permissions and exported components, and blocked entry points. A journey is a user goal or meaningful task, not a screen or test condition. Visiting a screen does not cover its actions or states. Font scale, display size, contrast, night mode, locale/RTL, orientation, window size, and similar conditions are never standalone JIDs.

Finder reports include only map deltas (new or corrected journeys, prerequisites, traps, blockers) and coverage deltas (below). Do not return a rewritten map. Broad discovery is exhausted only when the known frontier is empty and fresh runtime passes plus bounded source inspection reveal no new meaningful journey or high-risk branch.

## Coverage: hunt state in the ledger

Coverage is hunt state, so it lives in the host ledger (`ISSUES.md`), under a `## Coverage` heading, one pipe-delimited line per mapped journey:

```
J1 | coverage: discovered | tested: (none) | next: primary path through the naming offer
```

States: `discovered | attempted | core-covered | blocked | unsupported`. Conditions exercised (font scale, display size, locale/RTL, night mode, orientation, process death) go in `tested:` under the journeys they touched; they never substitute for journey coverage. A journey is `core-covered` only after its principal outcome and applicable state, cancellation, error, configuration-change, and persistence behavior have been observed.

The section is created at turn 1 with every mapped journey `discovered`, carried unchanged on resume, and never copied into `APPMAP.md`. Finder reports return coverage deltas as `J<n>: <state> | tested: +<condition> | next: <probe>`; the orchestrator applies them after each wave. To keep the host dashboard parser safe: coverage rows never carry a `status:` label, never sit in a markdown table, and never contain a literal `|` inside prose.

## Repository priors and precedence

Use `HUNT.md` or equivalent repository priors to specialize the skill, not to silently replace its purpose. Product facts, known intentional behavior, safety constraints, capability limits, build traps, proven dead ends, and defect-specific verifier conditions take precedence over generic skill assumptions.

Priors documents must carry a fixer-facing section: Gradle and test traps, CI gates (lint, detekt, ktlint, instrumentation tests), retry-chain tar pits with their commit hashes, and repo tooling a fixer needs are forwarded to fixers, never held as orchestrator-only. Tooling traps that produced a rejected candidate (tap latency, an empty or stale hierarchy dump, a leftover font scale or locale) belong in the section every finder receives. Cross-references to a prior hunt's issues use `round-N #M`; bare `#N` refers only to the current ledger.

Historical bug yield is assignment evidence, not a standing waiver of journey-first ordering. Track weights, wave-one mandates, or instructions that move accessibility-specific checks ahead of core UX/functionality apply only when the user or current host prompt explicitly requests that objective for this target. Imperative wording or yield statistics in a mined `HUNT.md` do not make it current. Otherwise retain them as leads for the applicable escalation tier. When a prior conflicts with the current host objective, the current objective wins; record the conflict rather than combining contradictory instructions.

## Agent budgets - include this block verbatim in every subagent prompt

> **Budget.** Context cost grows with every turn you take. Work within these caps; never push past them. Inside them, use the budget: return early only when your assigned journey's probes and the leads you opened in it are exhausted, not once the obvious probes pass.
> - **Finders:** return after **about 30 emulator interactions or 3 candidate issues**, whichever comes first. Each emulator finder owns one primary JID and at most one adjacent branch; complete that journey rather than skimming several. If setup consumes most of the budget, return the exact checkpoint and remaining probes. If you cannot reach an assigned path within 10 actions beyond its documented setup, record it as blocked, naming the real gesture you tried and why it failed, and move on. If you can reach a state but cannot measure it (count objects on a GL surface, catch a short-lived state, read a timer), return `FIXTURE NEEDED: <log line or state hook that would show it>` rather than dropping the lead. Do not re-report the open root causes your prompt lists; a symptom of one of them is a note, not a candidate. Before returning, drive the gesture your journey names (swipe, drag, long-press) with real input such as `input swipe`, not a button that does the same thing, and spend what is left of the budget on the probes you have not reached. A state you can work out from data the app ships (a filter set that matches nothing, the smallest category) is computed with a script before it becomes `FIXTURE NEEDED`. Keep wrapper scripts and scratch files in your evidence directory, never in a shared `/tmp`. Before claiming something is missing from a screen that scrolls, scroll to its end.
> - **Hard stop:** at **60 tool calls total** (finders) or **40** (verifiers), stop and return the report with whatever you have, even mid-probe. Count every call. No finding justifies passing this line; a partial report is the expected outcome, not a failure.
> - **Setup cost:** if reaching a required state (a large arrangement, many records, a full library) would take more than **20 actions** beyond the documented reset, do not tap it into existence. Return `FIXTURE NEEDED: <state>` with the exact file or data shape, so the host can seed it (`adb push`, `run-as <package>`, a debug intent, or a snapshot the emulator boots from) before the next pass.
> - **Verifiers:** the defect repro itself must complete in **15 emulator actions or fewer**, excluding documented fixture setup. If it does not, return `BLOCKED: could not complete repro path` with what you tried. Do not keep exploring to make the repro work.
> - **Fixers:** return after **80 tool calls or 25 minutes** with the diff, the test result, and what is still failing. Do not keep iterating past that line; a partial fix with an honest status beats a silent overrun. Before changing code, check the evidence and `APPMAP.md`: if the evidence shows no defect, or the map marks the behavior as intended, commit nothing and return `NO DEFECT: <reason>` so the host can reject the issue. A fix for a misread repro changes intended behavior and has to be reverted later. Write the regression test first and run it against the unfixed code; report that failing run, then the passing one. While iterating, run only the new test class and the classes that cover the files you changed; run the full suite once, last, before the commit (a cold full suite can take 14 minutes under the machine's heavy-job lock). A test written alongside the fix may pass with the fix reverted. When `APPMAP.md` or the source names sibling triggers for the broken mechanism (`TIME_SET` and `TIMEZONE_CHANGED`, rotation and process death, each entry point to one dialog), fix and test every sibling, not only the one in the repro. After the commit, copy the smoke-tested APK to `<evidence-dir>/builds/<hash>/app-debug.apk` and report that path; the host installs only from there. `connectedDebugAndroidTest` uninstalls the app from the emulator when it finishes: reinstall the smoke-tested APK, restore `dumpsys battery unplug`, and say so in your report. List every user-facing string you add or change, with the screen it appears on. Messages say what happened and what the user can do, in plain words: never a raw system error, exception text, file name or path.
> - **Navigation:** use the assigned serial and the device server's find-element call to locate controls. A full hierarchy dump (the server's list-elements call or `uiautomator dump`) is allowed at most **once per new screen state**, never after every tap.
> - **Screenshots:** capture with `adb -s <SERIAL> exec-out screencap -p > <path>.png`, downscale a viewing copy before inspection (`sips -Z 750 <copy>` or `magick <copy> -resize 750x750 <copy>`), retain the original evidence, and view each image at most once. Take screenshots when the track's method requires them or to capture evidence, not as a general navigation aid.
> - **Evidence:** return one primary artifact per finding. A bounded before/after pair or short video/contact sheet is allowed when a single image cannot prove the claim.
> - **Report:** return a structured list with at most 3 candidates. For each: journey ID, preconditions, exact repro, expected basis, expected vs actual, user impact, reproduction count when practical, evidence, severity guess, and suspected duplicate. Add compact map and coverage deltas and worthwhile leads. Keep the report at **25 lines or fewer** and do not narrate process or dead ends.

The host sets the model explicitly on every dispatch (its default worker model for finders, verifiers, and first fixes; its escalation model for refixes). A dispatch that leaves the model unset has not applied the host's model policy.

## Severity ladder

Assign severity from demonstrated user impact, not from the finder track or checklist category:

- **S1 - catastrophic:** crash or ANR on a consequential or normally reachable path, unrecoverable data loss, security/privacy exposure (including an exported component or intent that leaks data or performs privileged actions), or a severe unintended irreversible action.
- **S2 - task failure or harmful result:** a meaningful user goal is blocked; committed or displayed data is wrong; input is lost, including across rotation or process death where the app is expected to preserve it; an action produces duplicate or unintended effects; recovery requires abandoning the journey, clearing app data, or repairing data outside its normal flow.
- **S3 - material UX failure:** the task remains possible, but the app misleads the user, obscures necessary state, behaves inconsistently enough to cause a wrong action, or cannot be operated reliably. Examples include indistinguishable loading/error/empty states, absent feedback that leaves a consequential action ambiguous, keyboard (IME) obstruction, content under system bars, system Back that exits or discards unexpectedly, meaningful truncation, recoverable lost progress, or an unreliable control.
- **S4 - polish:** visible inconsistency or aesthetic degradation that does not obscure meaning, change behavior, mislead the user, or reduce reliable operation.

Record reproduction confidence, journey importance, recoverability, and affected conditions separately from severity. A rubric match is not automatically S3: state the actual user consequence. The host goal may count S1-S3 plus at most two S4 findings. Deduplicate by demonstrated behavior and triggering condition first; merge by root cause only when evidence supports it. A sibling trigger that an earlier fix missed (same mechanism, different system event) shares that issue's root: it is not a duplicate (the earlier fix did not cover it), so it is never REJECTED as one: file it with `duplicate/root: root of #N`, fix it, and count the root once in the tally.

## Candidate and evidence rules

Track A, B, and D candidates require runtime evidence from the installed build. Reproduce from a known baseline twice when practical; record results such as `2/2` or `1/3`. One clear reproduction may be sufficient for destructive S1 behavior that is unsafe to repeat. Tool failure, environment contamination, unsupported capabilities, and uncertain product intent are not candidates.

Harness artifacts are a named false-positive class. Before any "silent", "no feedback", "nothing happened", "stuck", or "wrong state after tap" observation becomes a candidate, repeat it once with the lowest-latency tap available (`adb -s <SERIAL> shell input tap <x> <y>`, or the server's equivalent) followed immediately by `adb -s <SERIAL> exec-out screencap -p`, so a slow or misplaced automation tap cannot masquerade as an app defect. A `uiautomator dump` taken while animations run can be stale or empty; confirm against a screenshot before trusting it. An absence claim ("no warning", "the line is missing", "nothing explains it") on a screen that scrolls needs the screen scrolled to its end, with a dump or screenshot at each position: one viewport proves nothing about what is below it. A confirm agent re-checking an absence is told the exact text to look for and to scroll for it. On 2026-11-16 a finder and its confirm both reported a missing notification warning that sat just below the fold on the compact phone; only the fixer's evidence check caught it. Never estimate when a tap landed: if timing matters, bracket the tap and any recording with wall-clock timestamps. Environment contamination (a leftover locale, night mode, font scale, density, animation scale, demo mode, revoked permission, or dirty app data left by an earlier worker) is the other common source; when a candidate is rejected for either cause, record the root cause under the map's tooling caveats in the same turn so later finders inherit it.

An absence claim on a GL or canvas surface ("the sprite never appears") needs log or state evidence, such as a logcat line or a debug state dump showing that the object was never placed. Frames alone are not enough, because a 1 fps contact sheet can miss a brief appearance. When two workers independently hit the same interaction trouble on one control, the host treats it as a lead to prove either way (harness or app) before writing it off as tooling.

A journey is `blocked` for tooling only after the real user gesture was tried (long-press, lift, then tap, not a debug extra that fakes a finger) and failed for a reason the finder names. When the agent's screenshot-to-tap round trip can't meet a time window the app gives a real user (a control that stays up 5 seconds after the finger lifts), the finder returns `FIXTURE NEEDED: <debug hook that holds the state>` and the host builds the hook as a fixture task. On 2026-11-15 a free-a-koi branch was marked blocked after 11 taps through a fake-finger extra. A source audit showed the board stays put 5 seconds after a real lift; a retry then failed 4 of 4 on latency, and the round ended with no hook requested.

A candidate reaches CONFIRMED only with an independent reproduction: a separate confirm agent, or the finder's own `2/2` with evidence for a stable, deterministic defect. A fixer is never dispatched from a single unreproduced report; the false positives that cost the most are the ones that reach a fixer and then a verifier.

Use evidence appropriate to the claim:

- Stable visual or state defect: screenshot.
- Lost or inconsistent state: bounded before/after pair with the same sentinel data.
- Transient interaction defect: short video or contact sheet with timestamps.
- Crash or ANR: full triggering sequence plus the stack from `adb -s <SERIAL> logcat -b crash -d` (or the `ANR in <package>` block from `logcat -d`). A return to the launcher is a crash lead until the crash buffer, a process-exit line, or `dumpsys dropbox --print data_app_crash` supports it.
- Performance: commands, conditions, all measurements, and summary statistic.
- Source lead: file and line or symbol plus a proposed user-visible repro; no screenshot required.

A Track C finding may be a candidate without driving an emulator only when it includes file/line evidence and a precise user-visible repro that an emulator finder can drive with existing launch intents or extras. Otherwise it is a lead. Security, privacy, silent data-integrity, or background-behavior risks that cannot meet the user-visible gate remain leads with category-appropriate supporting evidence.

## Finder tracks

Hand each finder the budget block, an atomic `APPMAP.md` read instruction or full map, host priors that apply to it, and only its own track section. Before dispatch, cost the assignment against the budget: a probe whose setup alone exceeds the finder cap is a fixture task, not a finder task. Build the fixture (a seeded database, a pushed file, a debug-intent import, or a snapshot the emulator is relaunched from) with an off-emulator worker or a shell command, install it, and only then assign the probe. Never write a prompt that asks a finder to construct a large state by hand. Probes a finder prompt lists are a floor: the prompt says that once they are done, the finder continues down its coverage row's `next:` until the cap. On 2026-11-16 a finder given "probes to cover (3-4)" returned after 3 of its 12 minutes with card attach and a cross-tab delete untested, "outside my budget". Every track reports user impact, map deltas, and coverage deltas using the common rules above.

### Track A - Journey and UI functionality

Drive meaningful user goals through the tree. Coverage is verified state transitions, not controls or screens visited.

For each assigned journey:

1. Establish the documented baseline and unique sentinel data.
2. Perform the primary action and observe immediate feedback.
3. Wait for a settled success or failure state and verify the promised outcome.
4. Check other visible representations of the result, such as list/detail, badge/count, search/filter, widget, or notification.
5. Navigate away with system Back and in-app navigation and return; relaunch when persistence or restoration is part of the product contract.
6. Exercise applicable validation, cancel/Back, retry, destructive confirmation, repeated action, and interruption behavior.

Baked-in probes, applied only when relevant:

- Rapidly repeat consequential submit, save, purchase-sandbox, like, or destructive actions; judge duplication against the action's contract rather than assuming every repeat is invalid.
- Enter empty, invalid, minimum, maximum, duplicate, realistic long, and repeated data where the UI implies such boundaries.
- Begin an edit and separately test cancel, system Back, and save. Check that each result appears consistently throughout the app.
- Once per stateful assigned area, rotate mid-flow when the screen supports rotation (`adb -s <SERIAL> shell settings put system accelerometer_rotation 0` then `settings put system user_rotation 1`, restore to `0`). Configuration change recreates the activity; entered text, scroll position, selection, open dialogs, and in-flight results that vanish are candidates.
- Once per stateful assigned area, test process death: background the app with `adb -s <SERIAL> shell input keyevent KEYCODE_HOME`, run `adb -s <SERIAL> shell am kill <package>`, then return through Recents. Prove the process died: `adb -s <SERIAL> shell pidof <package>` before the kill, empty after it, and a different PID after the return. `am kill` only kills a backgrounded process, and `am start` prints "current task has been brought to the front" even when the process was killed, so that line proves nothing. Android promises the user comes back to the same screen; look for lost input, crashes on restore, blank screens, or state that contradicts persisted data. `am force-stop` is a different contract (user-initiated stop); do not assume drafts must survive it unless the app promises restoration.
- If the app unexpectedly exits or freezes, read `adb -s <SERIAL> logcat -b crash -d` and search `logcat -d` for `ANR in`, then replay the full sequence before classifying it as a crash or ANR.

### Track B - Visual-functional UX and consistency

Screenshots are the inspection method for this track. Navigate through actual journeys, then capture representative states rather than every screen. Inspect pixels together with observed interaction behavior.

Core rubric:

- **State clarity:** loading, empty, error, success, selected, disabled, destructive, and completed states are distinguishable and lead to a clear next action.
- **Action feedback:** consequential work produces perceivable, accurate feedback while in flight and at completion. A progress indicator or disabled button is not mandatory; optimistic updates, navigation, inline changes, snackbars, and system feedback can be valid. File a candidate only when the absence or inaccuracy causes demonstrated ambiguity or harmful repetition.
- **Behavioral consistency:** equivalent actions use consistent labels, placement, gestures, confirmation, cancellation, undo, and feedback. Compare equivalent concepts, not merely similar-looking controls.
- **Visual operability:** controls do not overlap, clip, shift out of reach, or become obscured by bottom sheets, the status bar, the navigation bar or gesture area, display cutouts, or the IME. Apps targeting API 35+ are forced edge-to-edge, so missing inset handling is a frequent real defect. Visible target size alone does not prove the touch area; verify unreliable activation behavior before assigning S3.
- **Content integrity:** realistic long content remains understandable; truncation or wrapping does not change meaning or hide required actions.
- **Cross-state integrity:** transitions, recomposition, and list updates do not flash stale or wrong data, lose identity, jump to an unrelated scroll position, or settle into an ambiguous state.
- **Appearance:** sample representative flagship states in light and dark theme. File issues when information or operation changes, not for preference alone.

```
adb -s <SERIAL> shell cmd uimode night yes
adb -s <SERIAL> shell cmd uimode night no
```

Capture IME-up evidence on input-heavy journeys and before/after evidence for state changes. Preserve full-resolution originals; inspect downscaled copies. Pure visual variation without user consequence is S4 or a product question.

### Track C - Source-assisted user-impact audit

Read source without driving an emulator. Use `APPMAP.md`, the manifest, and navigation graphs to connect implementation risks to mapped journeys and to discover hidden capabilities, routes, prerequisites, and state branches.

Prioritize:

- Wrong data, boundary, date/timezone/locale, sorting/filtering, and persistence logic, including Room migrations and DataStore/SharedPreferences writes.
- Errors that are discarded, converted to false success, or leave loading state stuck; `CancellationException` swallowed by broad `catch`, or `runCatching` around suspend calls.
- Races, duplicate requests, unscoped or `GlobalScope` coroutines, stale responses, collection outside the lifecycle, and shared mutable state.
- One logical change split across multiple state systems, especially ViewModel state plus Compose `remember` state or Fragment fields, and state held outside `SavedStateHandle`/`rememberSaveable` that the user expects to survive rotation or process death.
- `!!`, uninitialized `lateinit`, unchecked casts, platform-type nulls from Java APIs, ignored write results, and user-controlled indexing.
- Main-thread disk or network work, `runBlocking` on the main thread, and heavy work in `onCreate` or composition that risks ANRs.
- Authentication/session, role, entitlement, exported activities/services/receivers/providers, `PendingIntent` mutability, deep-link intent filters, import/export and content URIs, WorkManager/background work, notifications, runtime permissions, and declared capability paths applicable to the app.
- Silent security, privacy, analytics-consent, background-sync, backup (`allowBackup`, data extraction rules), and data-integrity failures.

A lead includes file/line or symbol, affected journey, trigger, predicted user consequence, and a concrete runtime probe. Ordinary leads become candidates only after user-visible runtime reproduction. Return hidden-route and prerequisite discoveries as `APPMAP.md` deltas. Pin reports to the source revision supplied by the host when fixes may occur concurrently.

### Track D - Targeted performance probe

Run after mapped journeys identify high-frequency interactions, visible stalls, jank, expensive screens, or repeated flows worth measuring. Emulator results are triage evidence, not physical-device certification. Measure a release-like build (minified, non-debuggable, or with a baseline profile) when the host can provide one; debug builds exaggerate cost, especially for Compose.

For interaction and launch timing:

- Define an observable start event and semantic completion condition before recording.
- Keep build variant, emulator, fixture, cache/network state, animation scales, and concurrent host load stable.
- Warm up once, collect at least three measured runs, and report all values plus median and worst case.
- Treat visible unresponsiveness over 1 second or cold launch over 2 seconds as investigation thresholds, not automatic defects. An ANR (input not handled within about 5 seconds) is a Track A S1/S2 finding, not a performance metric.
- Scene-change detection may locate a window to inspect, but scene counts and gaps alone do not prove a hang.

```
adb -s <SERIAL> shell am start -W -S -n <package>/<launcher-activity>
adb -s <SERIAL> shell dumpsys gfxinfo <package> reset
adb -s <SERIAL> shell dumpsys gfxinfo <package>
adb -s <SERIAL> shell screenrecord --time-limit 30 /sdcard/<unique-name>.mp4
adb -s <SERIAL> pull /sdcard/<unique-name>.mp4 <unique-path>.mp4
ffmpeg -i <unique-path>.mp4 -vf "select='gt(scene,0.02)',showinfo" -f null - 2>&1
```

`am start -W -S` reports `TotalTime` for a cold start. `gfxinfo` reports janky-frame counts and percentiles for the interval since reset. `screenrecord` captures at a variable frame rate; do not derive frame timing from it alone.

For repeated-flow memory investigation, warm the flow, sample a settled baseline, then sample after repeated batches such as 10x, 20x, and 30x with an idle control. Growth must remain material across batches and be attributable to the app-owned flow; one before/after increase, or a retained-object report without a user-reachable repeat path, is inconclusive. A retained-activity claim needs a forced GC before the count: LeakCanary in the debug build, or a debug-only hook that calls `Runtime.getRuntime().gc()`. Shell `kill -10` on an app process is refused, and `am send-trim-memory` does not force a collection, so `dumpsys meminfo` activity counts without a forced GC are inconclusive. The fixer's regression test must hold the real leaking object, not a stubbed wrapper around it.

```
adb -s <SERIAL> shell dumpsys meminfo <package>
adb -s <SERIAL> shell am dumpheap <package> /data/local/tmp/<unique-name>.hprof
```

## Wave composition

Before wave 1, ensure `APPMAP.md` contains at least install/launch/reset instructions, global navigation, known journeys, prerequisites, and test data. It may be source-derived initially. Runtime discoveries refine it.

Also before wave 1, read the tooling traps in `APPMAP.md` and `HUNT.md`. Every trap that names a missing hook, fixture or snapshot (a state that outlives a real finger by seconds, a widget host the renderer does not draw, a large data set) gets built by a setup worker before any probe that depends on it is assigned, and the host adds its `APPMAP.md` entry in the turn the hook merges. A trap is known cost: on 2026-11-16 a board-tap latency trap recorded the round before cost 4 failed confirms and an inconclusive smoke test until the hook existed, and a stale widget snapshot cost a finder 25 actions; each hook took a setup worker about 3 minutes. Before a Tier 1 wave that includes pseudo-locales, check that the debug build type sets `isPseudoLocalesEnabled = true`; if it doesn't, a setup worker adds it first. On 2026-11-16 the pseudo-locale check was skipped because the debug APK lacked them. Also check that the UI strings come from resources: when they are hard-coded, `en-XA` changes only system-formatted text such as dates, so skip it, keep `ar-XB` for mirroring, and record that in the map. On 2026-11-16 Hedgebook's setup enabled pseudo-locales, and `en-XA` then changed only the dates.

Before wave 1 and after every worker, assert each emulator's environment baseline and restore it when it drifts: locale and per-app language, night mode, font scale, display density and size, animation scales, navigation mode, rotation lock, demo mode, power state, and any permission grants a worker changed. Emulators report AC charging by default, so behavior that only runs while plugged in (screen-on modes, screensavers, deferred work) is active in every run. Unplug with `adb -s <SERIAL> shell dumpsys battery unplug` unless the probe is about charging. The baseline is unplugged, so a worker restores it with `unplug` again; `dumpsys battery reset` puts the emulator back on AC. On 2026-11-15 an app's charging screensaver faded its controls in every finder run until turn 18. The first launch on a fresh baseline shows the system "Viewing full screen" cling over the app. Suppress it with `adb -s <SERIAL> shell settings put secure immersive_mode_confirmations confirmed`. Record the baseline commands in the map's launch/reset block. Apply them before saving the snapshot. Never `snapshot load` on a running emulator. With `-gpu host` on emulator 37.2.13 it segfaults the emulator a few seconds later, in host GL cleanup (`EglConfig::compatibleWith`) or the adb pipe (`AdbVsockPipe::setSocket`): six crashes in 11 hours on 2026-11-15 and 16, across two apps' hunts. A snapshot is for boot only, and it holds the APK installed when it was saved. The reset between workers is app-level: `am force-stop <package>`, then the app's own debug wipe (a launch extra) or `pm clear <package>`, then the environment-baseline commands. Record that exact reset in the map's launch/reset block. The host owns re-baselining and may run it itself without a setup worker: after any commit that changes the APK (a fix, a refix or a hook), `install -r <evidence-dir>/builds/<hash>/app-debug.apk` on every hunt emulator and assert the environment baseline. After every worker, the host checks that each emulator is alive (its `qemu` process and `adb -s <SERIAL> get-state`). A dead one is relaunched from `<app>-baseline`, gets the current build and the asserted baseline, and only then gets work. On 2026-11-16 a fixer found both emulators dead only when its install failed, and its fix was committed without a smoke test. Never install from `app/build/outputs/apk/`: a fixer building in the shared checkout overwrites it with unfinished work, and on 2026-11-16 the host installed a fixer's half-done APK that way. A worker that inherits a contaminated emulator produces false positives that look exactly like defects.

Every worker that changes device state restores it before returning: font scale, density, night mode, locale, rotation lock, `dumpsys battery unplug`, `screen_off_timeout`, and after any clock or timezone probe `settings put global auto_time 1` and `auto_time_zone 1` plus a `date` check that the clock and timezone match the host's (`getprop persist.sys.timezone`). Restore by command, never by `adb unroot` or a snapshot load.

One worker drives a serial at a time. When a fixer's smoke test needs an emulator a finder holds, the host takes the serial with a no-clobber lock file (`set -C; echo <worker> > <evidence-dir>/locks/<SERIAL>.lock`). Workers delete their lock as their last step, only after the restore succeeded, never inside a command chain that can fail first (on 2026-11-16 a shell error mid-chain still deleted a setup worker's lock). A dispatched worker's cap starts when it gets the lock. A lock whose worker has returned is stale: the host asserts the baseline, then removes it. A fixer's dispatch prompt names the one serial it may smoke-test on, and the host sends no finder to that serial until the fixer returns. A fixer told to take whichever emulator frees first races the host's next dispatch: on 2026-11-16 that happened twice, and each time a finder went out to an emulator the fixer had just locked.

The host enforces every worker's cap. It records each dispatch's start time and cap, and wakes at the earliest running worker's cap plus 5 minutes; that wake is not a heartbeat. A worker still running then is overrunning: the host asks it once, by its task ID, to stop and return its partial report within 2 minutes, then stops it. The host removes its lock, reverts only that worker's uncommitted files, asserts the emulator baseline, and records the attempt in the issue row; a fixer stopped this way counts one stuck turn. Waiting on completions alone misses an overrun, because a worker that keeps resuming its own background work sends interim notices, not a report. On 2026-11-16 a Hedgebook fixer ran about 50 minutes against a 25-minute cap, mostly in full-suite runs holding the machine's heavy-job lock, while an S3 fix waited behind its S4.

Assign one emulator-bound finder per free emulator, and name the one serial it may drive. With the host protocol's two emulators:

- **EMU-A:** Track A tests and expands one highest-value mapped journey.
- **EMU-B:** Track B inspects one different journey's representative states and screen family for operability and consistency.
- **Off emulator:** one or two non-overlapping Track C auditors inspect different risk areas or source partitions.
- **Track D:** interleave only after an emulator frees up and a named journey or symptom warrants measurement.

Merge map and coverage deltas after the wave, then assign later waves from uncovered journeys, states, and applicable risks in the ledger's coverage section. Give fresh agents a short summary of prior findings, known dead ends, and their exact coverage gap. Every finder prompt carries one line listing the open root causes (CONFIRMED, FIXED-UNVERIFIED, and this round's VERIFIED) with "don't re-report". A next-probe that only re-tests an open root is not a finder assignment: send it to that issue's fixer as scope, or to its verifier as the adjacent check. On 2026-11-16 a finder sent at an open root's journey came back with a duplicate of it. Never spawn agents merely to keep an emulator busy, but while a fixer or verifier occupies one emulator and the host's spawn condition holds, keep the free emulator on the next uncovered journey rather than idle.

Treat dispatch as atomic: build and inspect the complete prompt before launching the worker, then retain its task ID. If a running worker needs corrected context, message that exact task ID. Never launch a separate "correction" agent. If messaging is unavailable, stop the malformed worker and launch one replacement. After dispatching the wave, emit one compact dispatch delta and stop; do not repeat status narration while workers run.

## Escalation

A wave is dry for a track when it produces no new distinct-root-cause candidate, as defined by the host protocol. Coverage deltas still direct the next assignment but do not count as candidates. A dry pass on one JID does not exhaust Tier 0 while another journey in the ledger's coverage section remains `discovered` or `attempted`; assign the applicable Track A/B finder to that uncovered journey next. Advance beyond Tier 0 only after the core-journey frontier is `core-covered`, `blocked`, or `unsupported`. A journey whose finder returns dry with only edge-case next-probes left (a boundary, a rare timing, a second entry point to a covered outcome) is `core-covered`; keep the leftovers in `next:` as leads. When every journey still uncovered is already assigned to a running finder, a free emulator starts the next tier's wave rather than idling: on 2026-11-16 a compact phone sat idle at turn 34 until a stop hook prodded twice. A newly discovered journey returns applicable work to Tier 0. After that gate, never repeat an escalation tier that came up dry.

When one mechanism has two CONFIRMED or VERIFIED issues (state lost on process death on two screens, a missing debounce on two buttons), a third instance doesn't go to another finder-confirm-fix-verify cycle. A Track C auditor lists every remaining instance in source, a confirm agent checks them on the emulator, and the ones that reproduce become one issue with one fix and one verifier who checks each of them. On 2026-11-16 Explore search and onboarding picks each took a finder, a confirm, a fixer and a verifier for the same missing `SavedStateHandle` state, and a third process-death loss (category scroll) took a third cycle. The sweep also catches instances like that one, whose cause turned out different: the list composed before its data loaded, so the restored scroll was clamped. Each different cause stays its own issue.

Run one wave per tier, then escalate as the host protocol requires. Diversify categories when a wave has multiple finders, risk-rank them for the actual app, and explicitly skip irrelevant categories rather than running generic checks blindly.

### Track A ladder

**Tier 0 - functional depth:**

1. CRUD or equivalent lifecycle: create, reopen, edit/cancel, save, delete/undo, relaunch.
2. Validation and recovery: invalid to corrected input without losing unrelated work.
3. Cross-view consistency: list/detail, counts, search, filter, sort, and stale refresh behavior.
4. Interruption and concurrency: repeated actions, Back/cancel during work, rotation, background and process death, stale responses.
5. Empty, populated, maximum, duplicate, partial, and failed states applicable to each core journey.

**Tier 1 - cheap applicable conditions:**

1. Runtime permission denial, "don't ask again", and one-time grants for permissions the app actually requests (Android 13+ includes `POST_NOTIFICATIONS`):
   ```
   adb -s <SERIAL> shell pm revoke <package> android.permission.<NAME>
   adb -s <SERIAL> shell pm reset-permissions -p <package>
   ```
2. Declared deep links and app links, each cold and warm:
   ```
   adb -s <SERIAL> shell am start -W -a android.intent.action.VIEW -d "<url>" <package>
   ```
3. Declared notification routes. Tap handling is usually an activity intent with extras; drive it directly with sanitized values, or post through the app's debug hook or an FCM test send when one exists:
   ```
   adb -s <SERIAL> shell am start -n <package>/<activity> --es <key> <unique-value>
   ```
4. Date and timezone boundaries where the app displays, groups, expires, or calculates dates. Requires a root-capable image; restart the app afterward:
   ```
   adb -s <SERIAL> root
   adb -s <SERIAL> shell settings put global auto_time_zone 0
   adb -s <SERIAL> shell setprop persist.sys.timezone Pacific/Kiritimati
   ```
5. "Don't keep activities" as a cheap amplifier for restore bugs on every back-stack return:
   ```
   adb -s <SERIAL> shell settings put global always_finish_activities 1
   adb -s <SERIAL> shell settings put global always_finish_activities 0
   ```

Force-stop before applying launch-only conditions. Restore and verify the baseline after each assignment.

**Tier 2 - changed environment:**

6. Locale and RTL when localization is supported; find navigation and meaning failures, not untranslated-copy inventory. Per-app language (Android 13+) avoids touching the system locale:
   ```
   adb -s <SERIAL> shell cmd locale set-app-locales <package> --locales ar-SA
   adb -s <SERIAL> shell cmd locale set-app-locales <package> --locales ""
   ```
7. Offline, slow, or lossy networking. Emulator network controls are per-instance and do not touch the host:
   ```
   adb -s <SERIAL> shell cmd connectivity airplane-mode enable
   adb -s <SERIAL> shell cmd connectivity airplane-mode disable
   adb -s <SERIAL> emu network speed gsm
   adb -s <SERIAL> emu network delay umts
   ```
8. Memory pressure without full process death:
   ```
   adb -s <SERIAL> shell am send-trim-memory <package> RUNNING_CRITICAL
   ```
9. Multi-window, role/session expiry, import/export through the system picker and share sheet, media and audio-focus interruption, or hardware-related states only when supported and automatable.

**Tier 3 - disruptive or manual:**

10. Large screens and foldables for apps that declare them: resizable or foldable AVDs, fold/unfold posture changes, freeform windows. Mark unsupported coverage honestly.
11. Background restrictions: Doze and App Standby for apps with scheduled or background work (`adb -s <SERIAL> shell dumpsys deviceidle force-idle`, restore with `unforce`), and physical-device-only capabilities.

**Tier 4 - accessibility-specific audit:**

12. TalkBack meaning and focus order, content descriptions, roles/states/actions, live-region announcements, Switch Access, and accessibility-specific system settings. This is escalation-only unless the host requests it earlier. Grade defects with the same user-impact ladder.

### Track B ladder

**Tier 0 - representative state coverage:**

1. Compare equivalent controls and components across core journeys.
2. Capture validation, IME-up, loading, error, empty, success, destructive, and completed states where applicable.
3. Sample light/dark and the smallest supported window on flagship states. Display size changes density-based layouts:
   ```
   adb -s <SERIAL> shell wm density <dpi>
   adb -s <SERIAL> shell wm density reset
   ```
4. Exercise realistic long and boundary content using app-valid data.

**Tier 1 - applicable compatibility conditions:**

1. Pseudo-locales for fixed-width and meaning failures, when the debug build sets `pseudoLocalesEnabled true`. `en-XA` lengthens and accents strings; `ar-XB` mirrors layout:
   ```
   adb -s <SERIAL> shell cmd locale set-app-locales <package> --locales en-XA
   adb -s <SERIAL> shell cmd locale set-app-locales <package> --locales ar-XB
   ```
2. Status-bar extremes where layout uses the top inset:
   ```
   adb -s <SERIAL> shell settings put global sysui_demo_allowed 1
   adb -s <SERIAL> shell am broadcast -a com.android.systemui.demo -e command clock -e hhmm 2359
   adb -s <SERIAL> shell am broadcast -a com.android.systemui.demo -e command battery -e level 4 -e plugged true
   adb -s <SERIAL> shell am broadcast -a com.android.systemui.demo -e command exit
   ```
3. Three-button versus gesture navigation, for bottom inset and Back handling:
   ```
   adb -s <SERIAL> shell cmd overlay enable com.android.internal.systemui.navbar.threebutton
   adb -s <SERIAL> shell cmd overlay enable com.android.internal.systemui.navbar.gestural
   ```
4. Supported orientation, display cutout, and tablet or window-size-class changes.

Force-stop before launch-only conditions and restore the baseline after each assignment.

**Tier 2 - transient motion, symptom-driven:**

Record only interactions with a visible flash, identity swap, stale frame, jump, or ambiguous completion. Use scene detection to locate inspection windows, then judge explicit invariants; do not use scene-change counts as a defect threshold.

```
adb -s <SERIAL> shell screenrecord --time-limit 15 /sdcard/<unique-name>.mp4
adb -s <SERIAL> pull /sdcard/<unique-name>.mp4 <unique-path>.mp4
ffmpeg -ss <t> -t 1.2 -i <unique-path>.mp4 -vf "fps=30,scale=360:-1" <unique-dir>/f%02d.png
ffmpeg -i <unique-dir>/f%02d.png -frames:v 1 -vf tile=4x5 <unique-path>-sheet.png
```

**Tier 3 - accessibility-specific presentation audit:**

Largest font scale, high-contrast text, color correction, and removed animations run here unless requested earlier. Android 14+ scales text nonlinearly up to 2.0. A standards mismatch becomes a candidate only with category-appropriate evidence and severity based on impact.

```
adb -s <SERIAL> shell settings put system font_scale 2.0
adb -s <SERIAL> shell settings put system font_scale 1.0
adb -s <SERIAL> shell settings put secure high_text_contrast_enabled 1
adb -s <SERIAL> shell settings put secure high_text_contrast_enabled 0
adb -s <SERIAL> shell settings put global animator_duration_scale 0
adb -s <SERIAL> shell settings put global animator_duration_scale 1
```

### Track C and Track D escalation

Track C runs once per non-overlapping source partition and re-runs when new map leads or post-fix code warrant it. Track D runs once per named journey and metric; re-run only after fixes touch the measured behavior or prior measurements are inconclusive for a correctable reason.

## When to hunt and when to stop

A round is one hunt against one ledger (the `round-N` of priors); a wave is one dispatch inside it. The host's target count is a ceiling, not a quota.

- **When to hunt:** start a round for a release candidate or a feature that just landed, not as a repeat on an app with no release date. Before wave 1, read the last two rounds' verified severities from `HUNT.md` or their ledgers. If every verified fix in both was S3/S4, tell the user once that shipping beats another round, and repeat that recommendation in the final report.
- **Stop rule:** once the Tier 0 gate in "Escalation" is met, the first finder wave at a new tier that confirms no S1 or S2 issue ends finding. An S3 or S4 that wave confirms is fixed and verified, but it does not extend the round: "dry" is the escalation test, not this one. On 2026-11-16 Hedgebook's host read "nothing above S3" as "nothing at S3 or worse", and the goal hook then disputed a correct stop after a wave that confirmed one S3 and one S4. In round 1 (no earlier `round-N verified` line in `HUNT.md`), that wave is at Tier 1, with a Track A and a Track B finder, so a first hunt always gets one wave past Tier 0. If the gate is still unmet at half the host's turn cap, in any round, run a Tier 1 wave then and return to Tier 0 afterwards. The same applies when an idle emulator starts Tier 1 early (see "Escalation"). Each Tier 1 finder takes a `core-covered` journey as its primary JID and runs its condition there. A tier's wave is a Track A and a Track B finder, and it is judged only once both have returned, never on one track's half. A Tier 1 wave run before the gate that confirms no S1 or S2 is the stop rule's wave: finding ends as soon as the gate is met, with no further tier. A bar as high as `core-covered` across a dozen journeys rarely closes inside a turn cap: on 2026-11-15 an Android round ended at turn 40 with 12 of 13 journeys `attempted`, and no Tier 1 condition was ever tried. A re-audit of fixed code, a re-check of a covered journey, or a probe of a fix's own risks is regression work and never counts as that wave. When it ends finding, dispatch no more finders, carry CONFIRMED and FIXED-UNVERIFIED issues through fix and verification, then file the final report. Do not climb further tiers to reach the count. On 2026-11-09 Hedgebook's first round stopped at turn 26 of 40 after a dry wave that only re-audited the fixes, and no Tier 1 condition was ever tried.
- **Attempt cap:** the host's two-rejection rule is final for the round. A fix that fails verification twice becomes `BLOCKED`, with each attempt's approach and failure in its row, and goes to the priors' Fixers tar pits at wrap-up. No third or fourth attempt this round, on any model.
- **Ledger hygiene:** ledger, coverage, and map edits, including the map's revision restamp after each fix install, stay in the working tree and land in one commit when the round closes, with the `HUNT.md` append. Never commit per restamp, note, or status change; fixers stage only their own files.
- **New copy:** strings that fixes added or changed are placeholders until the user approves them. The final report lists each one with its screen and fix hash, and when the user has a copy-review flow, the host sends them all there as one review with screenshots. On 2026-11-14 an iOS hunt's fixers committed seven messages, two of them showing a raw system error and a file name.
- **Final report:** state what ended the round (count reached, turn limit, or stop rule), and append `round-N verified: S1 <n>, S2 <n>, S3 <n>, S4 <n>` to `HUNT.md` so the next round can apply the when-to-hunt check.

## Blind verification addendum

Verifiers receive the defect description, exact repro, expected behavior, target APK path and variant, `APPMAP.md`, assigned serial, and required environment or measurement procedure. Explicitly forbid reading source code, diffs, git history, commit messages, or implementation notes. All verdict evidence comes from driving the installed build.

The host hands the verifier a neutral copy of the fix's APK (`<evidence-dir>/verify/<issue>.apk`), never the `builds/<hash>/` path, which names the fix commit. For the same reason the verifier gets the map without its installed source revision line: paste the map with that line removed rather than pointing at the file. Before replay, record the tested APK path and build identity (`adb -s <SERIAL> shell dumpsys package <package> | grep versionName`), variant, emulator serial, API level, fixture/account, and relevant condition. Restore the original preconditions. When the defect or the fix depends on the build type (a debug-only path, an exported-component guard, minification), verify on that build type. A release APK over a debug install fails on the signature mismatch, so uninstall first on the hunt emulator only. Use a fresh mid-tier-or-better emulator agent and the budget block's verifier cap.

Allowed verdicts align with the host ledger:

- `VERIFIED`: the original defect is absent and the expected user outcome is observed.
- `REJECTED`: the original defect or an equivalent harmful behavior remains.
- `BLOCKED`: a prerequisite or repro path cannot be completed within the cap.

These are verifier verdicts. A verifier's `REJECTED` means the attempted fix failed and the confirmed issue returns for another fix; it is distinct from rejecting an unconfirmed candidate as intentional, duplicate, unsupported, or false positive. If the path completes but evidence cannot distinguish fixed from unfixed, return `BLOCKED: inconclusive evidence` and explain what is missing. For `VERIFIED`, rerun the exact repro, describe the observed state at the former defect location, and check one named adjacent invariant such as cancellation, repeated action, rotation, process death, persistence, or the equivalent control elsewhere. When the map names a sibling trigger for the same mechanism (a timezone change next to a clock change), the host names that sibling as the adjacent invariant. On 2026-11-16 a clock-backward widget fix passed verification, and the timezone change through the same path was filed and fixed as a separate issue.  For a scroll or layout fix, the adjacent invariant is that whatever the user is answering or reading there (the question, the prompt, the header text) is still fully on screen after the fix, checked at the smallest supported width. On 2026-11-03 a Fretalus fix that scrolled to the revealed board pushed the question under the header, the verifier passed it, and Chris found it after the hunt.

- **Track B fixes:** capture fresh evidence appropriate to the claim and describe the relevant pixels and behavior. "Looks fixed" is not a verdict.
- **Track D fixes:** repeat the same measurement under comparable conditions and quote all runs, median, and worst case against the original values.
- **Timing pitfall:** the harness-artifact rule in "Candidate and evidence rules" applies to verifiers too. Use `adb -s <SERIAL> shell input tap` for timed taps, bracket the tap and recording start with wall-clock timestamps, and calculate the tap offset. `screenrecord` start-up adds latency, so start it before the timed action. If timing worsens unexpectedly, investigate measurement error, a debug build, and host load before claiming regression.

## Notes

- Keep environment changes scoped to the assigned serial and restore baseline settings after each worker. `settings`, `wm`, `cmd uimode`, and `setprop` changes persist across app restarts and survive until reset or snapshot load.
- Accessibility-specific auditing is late by design for this workflow; observable usability failures remain core regardless of which users they affect.
- Escalation order reflects likely defect yield and execution cost, not possible severity. Rotation and process death sit in Tier 0 because they are the most common source of Android state-loss defects.
- Re-check commands against the installed platform-tools, emulator, and system image API level before relying on them; several `cmd` subcommands require Android 11 or 13+.
