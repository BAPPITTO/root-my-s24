# Contributing Guidelines

Thank you for your willingness to help. What this project needs most is **device/firmware adaptation**, followed by failure reports that actually include logs.

---

## Reporting Issues

Please use the repository’s **issue templates** directly (Bug / Success Report / Feature Request). The templates already list the required information. The following items are absolute prerequisites for troubleshooting:

- Full device model (e.g., `SM-S9280`)
- System version / One UI version
- **Full kernel version string** — `adb shell uname -r`. Provide the entire line, formatted like `6.1.145-android14-11-3254743-abS9280ZCS6DZH3`. **Do not just provide the trailing build number.** The same build number can have different offsets on different platforms.
- Build number (Settings → About phone → Software information, or `getprop ro.build.display.id`)
- App version + Engine (GhostLock 43499 / 98260 UAF) + KSU driver version (3.2.5 / 3.3.0)
- **Runtime log ID** (from v3.6.3 onwards: Log page → "Copy ID"), or paste the full log directly.
- The exact step where the failure occurred (the last `[*]` or `[-]` line in the log).

With this information, most issues can be diagnosed at a glance. For example:

> Kernel build number `33419968` differs from the adapted `3254743` → Payload cannot be reused; independent offset mapping is required.

> For the 98260 engine, you **must** specify "which attempt this is since the last boot": Repeated attempts within the same boot session will dirty the kernel state, and the log will halt midway through the `plant` phase.

---

## Device Adaptation (The Most Valuable Contribution)

This project uses two engines with different payload structures and adaptation processes:

| Engine | Applicable To | Payload Location | Mapping Basis |
| --- | --- | --- | --- |
| **GhostLock 43499** | Older firmware (6.1.145 and earlier) | `app/src/main/assets/cve-2026-43499-<build_number>` | Symbol table + p0 fingerprint |
| **98260 UAF** | One UI 9.0 / 6.1.162 and newer firmware | `app/src/main/assets/samtimer/targets/<TAG>/libtimer` | Symbol table + BTF + Call-site disassembly |

### 43499 Payload Adaptation

Even within the same SoC series, you **cannot** blindly reuse someone else’s payload. If the kernel build number differs, or even if the build number is the same but the platform is different, symbol offsets can vary (e.g., `e3q` and `e1q` both use `…-android14-11-3254743-…`, but testing shows 11 different offsets). Each offset must be mapped individually.

General process (refer to `docs/port-e1q-S9210ZCS6DZI1.md` for a complete example):

1. Extract `boot.img` from the official firmware package → kernel segment (note the boot header version and kernel segment offset).
2. Recover symbols using `vmlinux-to-elf`.
3. Map offsets in `target.h` item by item (`llvm-nm` + pointer scanning when necessary).
4. Verify `P0_KERNEL_PHYS_LOAD` — this item frequently requires device-specific adjustments (e.g., `0x80080000` will consistently stall at `p0 physical write status=256 ok=0`, whereas changing it to `0xa8000000` succeeds).
5. Generate the p0 fingerprint table.
6. Compile the payload.
7. **Real device verification.**

### 98260 Payload Adaptation (per-target, from issue #40)

The public payload for 98260 is `samtimer/libtimer` (covering EZIF/EZJ1). **Only add build-number-specific payloads; do not modify the public payload:**

```text
app/src/main/assets/samtimer/targets/<TAG>/libtimer   # Required: exploit payload compiled for this specific build number
app/src/main/assets/samtimer/targets/<TAG>/uhexec     # Optional: only include if it differs from the public version
app/src/main/assets/samtimer/targets/<TAG>/ksud       # Optional: only include if the public 3.2.5 driver fails to load
```

- `<TAG>` = **Full build number in UPPERCASE** (e.g., `S9280ZCS6DZJ3`. Do not use 4-letter abbreviations like `DZJ3`).
- Runtime selection is already implemented: if `samtimer/targets/<device_build_number>/` exists, it will be used; otherwise, it falls back to the public payload.
- Source code goes in `app/src/main/cpp/targets/<TAG>/` (offsets header + build instructions). Your PR must include:
  - **SHA-256** and file size of `libtimer`.
  - Build command (NDK version + flags).
  - Mapping record (symbol sources, which structures were verified via BTF, and failure signatures).

98260 mapping must cover at least the following:

1. The **return instruction address** of `kmem_cache_alloc(posix_timers_cache, …)` inside `do_timer_create()` (`*_STATIC` / `*_LOW21`), used for tracefs record matching.
2. cred/task allocation points in `prepare_creds()` / `dup_task_struct()`.
3. Static symbols: `selinux_state`, `panic_on_warn`, `system_unbound_wq`, `call_usermodehelper_exec_work`, `root_user`, `init_user_ns`, `anon_pipe_buf_ops`, etc.
4. Structure offsets (BTF verified): `k_itimer`, `signal_struct.posix_cputimers`, `workqueue_struct` / `pool_workqueue` / `pool`, `pipe_buf_ops`, `cred`, `task_struct`.
5. Constants related to kernel lines, such as `KERNEL_PHYS_LOAD` and `WQ_DFL_PWQ_OFF` (these differ between 6.1.145 and 6.1.162).

> Offsets can only be reused if the kernel build number (the `<build>` part in `6.1.145-android14-11-<build>-ab<BUILD>`) is identical. If the platform differs (e.g., `e3q` vs `e1q`), every offset must be verified individually, even if the build number matches.

### Hard Requirements for Submitting a PR

| Requirement | Reason |
| --- | --- |
| **Verified on a real device, with conditions stated** | Unverified payloads merged into the repo will be used by others on their actual devices. |
| Include payload **SHA-256 and file size** | Allows verification and prevents binary substitution. |
| Include a **complete offset comparison table** (vs. existing platforms in this project) | Helps future contributors identify patterns. |
| 98260 payloads must include **BTF / disassembly verification evidence** | Incorrect structure offsets don't just "fail to run"; they cause an immediate kernel panic. |
| Include **failure signatures** (the pitfalls you encountered) | Allows instant differentiation later between "payload mismatch" and "race condition didn't trigger". |
| New/modified payloads **must not contain** any external linking, downloading, or execution of third-party code. | This is a privilege escalation tool; user trust costs are extremely high. |
| Third-party `ksud` / module assets must cite source and license (MIT / Apache-2.0 / GPL). | This repository is GPL-3.0 and requires redistributability. |

A typical PR will need to modify the following areas:

```text
# GhostLock 43499
app/src/main/assets/cve-2026-43499-<build_number>     # Payload
app/src/main/kotlin/.../RootViewModel.kt              # Add new entry to FirmwareVersion
server/payloads.json                                  # Add new entry to online manifest
docs/port-<platform>-<build_number>.md                # Porting report

# 98260 UAF
app/src/main/assets/samtimer/targets/<TAG>/libtimer
app/src/main/cpp/targets/<TAG>/                       # Offsets header + build instructions
docs/port-98260-<TAG>.md                              # Mapping record
```

New entries should default to `tested = false` and only be changed to `true` after successful real-device execution.

---

## Self-Compiling and Self-Releasing (Forks)

**You may fork, but please read the "If You Are Forking and Compiling Yourself" section in `AGENTS.md` first.** 
Summary:

1. **Change the statistics upload address** (`LogUploader.DEFAULT_ENDPOINT`, `PayloadStatus.ENDPOINT`), or set it to `""` to disable uploading entirely.
2. **Use your own signature keystore.** The server distinguishes official from third-party builds by signature. Non-official signatures are marked as `third_party` and **excluded from statistics**. Therefore, forgetting to change the address won't pollute official success rates, but it will still waste official bandwidth.
3. **Custom payloads do not require server manifest changes:** Non-official builds automatically skip the online manifest and only use the built-in one.
4. **Do not blindly copy others' payloads.** Reasons stated above.

---

## Code Conventions

- Branch from `main`; PRs should target `main`.
- Commit messages must explain **why** the change was made, not just what was changed. Document any trade-offs considered.
- Comments should explain "why"; the code itself should explain "what".
- When adding new user-facing copy, simultaneously update `values-zh-rCN` / `values-zh-rTW` / `values-zh-rHK` (English goes in `values/`). Missing translations will gracefully fall back to English without crashing.
- After modifying app behavior, verify the target path on a real device at least once.
- When adding new AIDL methods, update both `IRootService.aidl` and the `SamTimerUserService` implementation simultaneously, maintaining backward compatibility (older UserService processes may still be running).

## What Will Not Be Merged

- **Unverified** new payloads (better to reject than to brick a user's phone).
- Changes that point statistics uploading to **third-party servers**.
- Removal of copyright/attribution notices, or code that conflicts with the project license.
- Noise PRs: version bump-only commits, empty commits, or pure formatting changes.

---

## Licensing

By submitting a contribution, you agree to release your contribution under this project's license. Please **do not** submit code of unknown origin, without a license, or that explicitly forbids redistribution (e.g., some repositories without a LICENSE file; you may reference their concepts, but do not copy their implementation).

Known redistributable reference implementations: SamTimer (MIT), Root-My-Galaxy-Payloads (Apache-2.0), KernelSU (GPL-2.0). Ensure corresponding declarations are retained in `THIRD_PARTY_NOTICES.md` upon merging.