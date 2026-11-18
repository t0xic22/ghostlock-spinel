# AGENTS.md — Developer Guide (English)

This file explains what the codebase does and the rules that must be followed when
modifying it. Every section describes the actual behaviour, not just a summary.

## What GhostLock does

GhostLock is an Android local privilege escalation (LPE) tool. It wins a PI-futex
race to overwrite a waiter object, then runs three write stages in order:

1. **W1** — disable SELinux enforcement.
2. **W2** — overwrite credentials to obtain `uid=0` (root).
3. **W3** — bypass seccomp.

Finally it hands off to a **root child** process that can load a KernelSU module.
A kernel is accepted only when its exact `uname -r` matches a bundled HOCON
profile. If no profile matches, the app refuses to run — this is intentional,
because the exploit depends on kernel-specific symbol offsets.

## Three-layer architecture

| Layer | Location | What it actually does |
|---|---|---|
| Android | `app/` | Kotlin/Compose UI; parses, merges and overrides HOCON profiles; hosts the Shizuku `UserService`; writes debug logs. |
| Native | `src/core/` | C++23 exploit runtime; builds to `build/native/ghostlock`. |
| Extractor | `tools/extract_rs/` | Rust tool that reads `boot.img`, an OTA package or a URL and emits either a flattened GLK1 profile (`--format conf`) or the legacy v1 JSON (`--format json`). |

Important details:

- The native binary is **not JNI**. `libghostlock.so` is a standalone executable
  ELF started by Kotlin with `ProcessBuilder`. Its CLI is
  `--ghostlock-app-call` (reads a length-prefixed GLK1 document from stdin) or
  `--load-prebuilt-profile <bin>`. The argument-less v1 `offsets.json` entry point
  was removed.
- Bundled profiles live in `app/src/main/assets/kernel_profiles/`:
  `index.conf` is the index, `<uname-r>.conf` is one file per kernel release,
  `execution-*.conf` holds shared/per-route tuning, and
  `credential-6x.conf` / `kernelsnitch-6x.conf` are shared 6.x fragments.
  The format is HOCON and supports `include`.
- The component model is **frontend × backend × middleware**:
  - frontend: `root_child` (the working one); `umh_forward` is an unusable placeholder.
  - backend: `cve_2026_43499` (working); `cve_2026_64560` is a placeholder.
  - middleware: `select_stack`, `tcp_zerocopy`, `multicast_waiter`.
  The combination is fixed at compile time by `Pipeline<F,B,M>`. The authoritative
  catalogue is `route/component_catalog.hpp`; selection comes explicitly from the
  profile/wire, never from the kernel version.

## Common commands

```sh
# Native layer
make -C src ghostlock          # NDK build (uses ANDROID_NDK_HOME or local.properties)
make -C src native-host-tests  # host unit tests (fixed vectors + lifecycle checks)
make -C src lint-tidy          # clang-tidy; must report 0 findings
ANDROID_NDK_HOME=... make -C src   # explicit NDK path when auto-detection fails

# Android / extractor
./gradlew :app:assembleDebug
./gradlew :app:testDebugUnitTest
./gradlew exportKernelProfiles  # writes GLK1 .bin files to build/kernel-profiles/
(cd tools/extract_rs && cargo test --release)

# Disassembly check — REQUIRED after any change to the attack path
python3 tools/cmp_disasm.py <baseline-binary> build/native/ghostlock
```

## Change workflow: design first, then code (mandatory)

Read `docs/development/design-philosophy.md` first. It contains 11 principles
grounded in SWEBOK / ISO 12207 / ISO 25010, with theory anchors and project examples.
The full engineering rules are in `docs/development/engineering-standards.md`.

Before **any** change:

1. **Explore** — read code and docs; use `git log --all -- <path>` for historical
   design decisions and device-gate evidence (`docs/analysis/**` lives in git history).
2. **Design** —
   - S-level (comments/formatting): edit directly.
   - M-level: document motivation, affected files, behaviour change and verification plan.
   - **L-level** (attack-critical path, wire/profile format, Native↔Kotlin contracts,
     public data structures, new routes): write a plan document first (template in
     `docs/development/documentation-standards.md`) and get user approval before coding.
3. **Implement** — smallest possible change; touch only files named in the design.
   Do not touch `kernelsnitch/`, the v1 conversion in `LegacyProfileConverter.kt`,
   or anything on the "explicitly preserved" list in the standards.
4. **Verify** — run the full gate for the change level and keep evidence. Report
   commands and results; never write "should be fine".

- Attack-path changes require all three: `cmp_disasm` (8 functions) + a real-device
  gate + a gate record (format in `docs/development/documentation-standards.md`).
- Large changes are delivered in batches: one concern per batch, next batch only
  after the previous one passes verification.

## Code conventions

- C++23, `-fno-rtti`, statically linked libc++; `-Wall -Wextra -Wconversion
  -Wsign-conversion`. New code must not introduce warnings. Includes use paths
  relative to `src/core` (e.g. `#include "route/tcp_zerocopy_route.h"`).
- Namespaces: `ghostlock::{route,session,memory,attack,race,support,profile}`.
  `kernelsnitch/` is verbatim upstream code — do not rewrite it.
- Only two globals are allowed: `g_exploit_session` (the single process-wide
  singleton) and `g_direct_map_end` (read-only after startup). No new mutable
  globals or reference aliases; see `docs/analysis/native-global-state.md` in git
  history for the audit.
- A component = identity (a declarative `kind` that must compile on the host)
  plus execution policy (frontend/backend steps, or a middleware route). Middleware
  policy uses compile-time capabilities and static hooks; backend steps are
  templates called directly on the middleware (no virtual base classes). A `Route`
  implements `prepare → execute → disarm → destroy` and reports only through `status`.
- Two-sided consistency (tests enforce both directions):
  - Native `profile/model.h` `RouteKind`/`kRouteCatalog` ↔ Kotlin
    `data/route/RouteKind.kt` (`route_catalog_test.cpp` / `RouteCatalogAgreementTest.kt`).
  - `profile/binary.cpp` field table ↔ Kotlin route config `entries()/apply()/from()`
    — key names must match exactly.
  - Route-private parameters go in the route extension section; only fields read
    by shared code go in the common slots (order matters there too).
- Configuration authority is the GLK1 profile + HOCON. The execution layer must
  not read configuration environment variables; only process/path ones are allowed:
  `GHOSTLOCK_HOME`, `TMPDIR`, `GHOSTLOCK_KSU_LOG`. New state must extend the profile.
- **Do not add or stack configuration/profile wire format version numbers.** This
  branch uses **v2** only: object sections `header + sections[name → fields[name → u64]]`,
  presence expressed by key existence, values stored bit-exact. Kotlin and native
  are version-bound and replaced together in one branch — no backward compatibility.
  If another branch needs a different format, change this document and
  `docs/development/` first; never start a v3/v4 in place.
- v1 (legacy `offsets.json`) is converted **only on the Kotlin side** by
  `LegacyProfileConverter.kt`; native no longer parses v1 (the JSON path in
  `src/core/legacy/` was deleted). Do not change the v1 conversion for new
  routes/fields.

## Core attack code review (only when touched)

Run this only when a change touches the core attack code or the code that prepares,
holds, accesses or recycles its resources. Ordinary UI, docs and config editor
changes are exempt. Core code includes waiter/race/payload/route/exec stages and
anything that directly manages their resources.

- **Ownership and lifetime tracking** — use Rust-style ownership reasoning: for
  every key object/pointer/resource list where it is created, who owns it, who
  borrows/accesses it, the access interval, the endpoint and who releases it.
  Confirm every user stops accessing before the endpoint; distinguish owning,
  borrowed, shared and transferred ownership, and verify the old owner does not
  free or access after a transfer.
- **UAF check** — walk every ordinary object and resource path and confirm there is
  no use-after-free. The only exception is a target deliberately used as the
  vulnerability primitive; never extend that exception to helpers, race state,
  waiters, buffers, mappings or synchronisation resources.
- **Endpoint and cleanup order** — verify destruction/recycling happens at sane
  lifecycle boundaries across success, failure, retry, cancel and early-return.
  In particular, preserve the existing order between PI memory/waiter access,
  synchronisation/detachment and reclamation. Past refactors that missed PI memory
  reclamation caused panics, so check cleanup locations and exit paths carefully.
- **Core markers and disassembly comparison** — mark the core attack functions and
  their prepare/hold/recycle code in the review record. After every refactor that
  touches core code, compare against the confirmed baseline with `tools/cmp_disasm.py`
  and disassemble the core functions plus their prepare/recycle code to verify
  machine-code order. The order between attack code and prepare/recycle must not
  change. If byte differences cannot prove that invariant, stop the batch and
  investigate — passing tests do not replace disassembly review.
- **Scope and evidence** — record the tracking table, diff conclusions and baseline
  ID in the batch plan or review record. Only the touched code and its lifetime
  closure need re-checking, but if that closure is attack-critical the full
  `cmp_disasm` + device gate + archive requirements still apply.

## Verification gates

- Ordinary change: `make -C src native-host-tests` + zero-warning NDK build +
  `make -C src lint-tidy`.
- Attack-critical change (waiter/race/payload/route/exec):
  1. `tools/cmp_disasm.py` across 8 attack functions — IDENTICAL (strict) or a
     reviewed annotated difference.
  2. Real-device gate: cold boot, fixed CPU pair, single route, clean start with
     KernelSU not loaded.
  3. Logs on the device at `Download/ghostlock-debug-log/<timestamp>/*.log.txt`
     (the same directory also holds `profile.conf`/`profile.bin`, recording the
     effective configuration and the exact GLK1 bytes sent to native). Confirm
     route hit and write verification.
- Archive device results in the format of `docs/analysis/device-gates/*.md` (a
  complete S/CPP/U01/NS* evidence set exists in git history).
- `KERNEL-PANIC-01` is a known environment/timing issue: the same build can pass,
  panic, then pass. Causality requires reproduction with the same build plus a
  cold-boot rerun — never blame code from a single panic.
- Current reality: Multicast (5.15), TCP and Select have all been verified on real
  devices by the original developers. A new profile must not be marked `supported`
  until it passes a real-device gate.

## Documentation conventions

- Full rules are in `docs/development/documentation-standards.md` (category map,
  naming/structure, Chinese/English writing, RFC 2119 wording, diagram rules,
  plan/gate templates, archive workflow, checklists; based on ISO/IEC/IEEE
  42010/15289/2651x, Diátaxis, DITA, Minimalism, Google style).
- **Structural and workflow changes must update the corresponding Mermaid/UML
  diagram.** L-level changes (routes, cross-layer contracts, session/resource
  ownership, attack stage machine, cleanup boundaries) require a diagram. Keep one
  authoritative diagram per structure and link to it from other docs.
- Bilingual upstream: `README.md` + `README_ZH.md`; every `docs/**` page had a
  `*_ZH.md` counterpart. **In this fork those Chinese duplicates were removed and
  the remaining documents standardised on English** — see `README.md`.
- Active docs: `README.md`, `docs/kernel_profiles/*`,
  `docs/development/adding-a-component.md`,
  `docs/development/design-philosophy.md` (read before changes),
  `docs/development/engineering-standards.md` (rules and gates),
  `docs/development/documentation-standards.md`, `src/core/README.md`.
- Historical docs were deleted from the working tree and can be recovered from git
  history with `git show <commit>:<path>`:
  `docs/analysis/**`, `docs/analysis/device-gates/**`, root `ARCHITECTURE.md`,
  `DECOUPLING_PLAN.md`, `DECOUPLING_LOG.md`, `PR_DESCRIPTION.md`, `RELEASE_NOTE.md`,
  `docs/pr-note-*`, `docs/release-note-v1.*`,
  `docs/development/native-modernization-plan.md`, `repro/xperia-first-success/`.

## Troubleshooting

- Gradle native build cannot find NDK: set `ANDROID_NDK_HOME` / `ANDROID_NDK_ROOT`
  or `ndk.dir` in `local.properties`.
- Rust cross-compilation for Android: use the rustup toolchain's `rustc` (Homebrew
  rustc lacks the `aarch64-linux-android` std) and
  `rustup target add aarch64-linux-android`.
- All generated artefacts belong under `build/`; do not commit `build/`, `*.so`
  or `local.properties`.
- Upstream submission flow: branch `very-not-stable-dev` → upstream `main`;
  commit messages use `type(scope): summary` (feat/fix/docs/refactor/test/chore).
  Do not commit or push unless asked.

## Fork-specific notes (this repository)

This fork runs the **CVE-2026-98260 timer exploit** instead of the upstream
`cve_2026_43499` backend, because the upstream route helper fails on this kernel.
Consequences:

- `app/src/main/jniLibs/arm64-v8a/libghostlock.so` is a prebuilt static ELF of the
  timer exploit. The Gradle tasks that build the C++ core and the Rust extractor
  are no-ops so the prebuilt binary is not overwritten.
- `app/src/main/assets/kernel_profiles/6.12.69-...conf` plus its `index.conf` entry
  exist only to satisfy the app's profile gate. The exploit has its own embedded
  profile and validates `uname -r` internally; the HOCON offsets are not used.
- `isKernelSupported()` returns `true` unconditionally for the same reason.
- `android:appComponentFactory="androidx.core.app.CoreComponentFactory"` is declared
  explicitly to work around a MediaTek/HyperOS bug in `LoadedApk.makeApplicationInner`.
- The app must run through **Shizuku** (uid 2000, `u:r:shell:s0`, Seccomp=0) because
  the exploit needs `tracefs` access that a normal app uid lacks.
