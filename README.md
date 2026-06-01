# fp00c9-standard-stack-port

Patch set, helper scripts, and activation notes for bringing the Synaptics `06cb:00c9`
match-in-sensor device onto the standard Linux fingerprint stack (`libfprint` + `fprintd` + PAM).

## Why this matters

Many laptops with Synaptics `06cb:00c9` match-in-sensor hardware do not have an obvious path to first-class Linux fingerprint authentication through the standard stack.
This repository documents one validated bring-up path from experimental driver analysis to `libfprint`, `fprintd`, and PAM integration.
The goal is to turn a local hardware fix into a reproducible, sanitized starting point for upstream review and other maintainers.

## What this repo contains

- `patches/libfprint-standard-stack.patch` — libfprint changes that enabled and stabilized the sensor.
- `patches/fprintd-standard-stack.patch` — fprintd changes for local testing, pairing persistence, and template fallback.
- `patches/pydrv-analysis-fixes.patch` — Python-driver fixes that informed the libfprint port.
- `scripts/` — build helpers plus local smoke-test wrappers used during bring-up.
- `docs/` — notes for local validation, system activation, and reproduction planning.
- `docs/REPRODUCTION.md` — repeatable workspace setup, build, and validation checklist for future maintainers.

## Deliberately excluded

This export is sanitized. It does **not** include:

- enrolled fingerprint templates
- pairing blobs / persistent sensor state
- `~/.local/state/fp00c9` runtime data
- `/var/lib/fprint` contents
- raw verification or enrollment logs that contain live biometric session data
- build artifacts (`builddir`, `stage`, `fprintd-stage`, `.venv`, etc.)

## Verified outcome

The working build described here reached:

- local standard-stack enroll: **completed**
- local standard-stack verify: **matched**
- system `fprintd-list`: enrolled `right-index-finger`
- system `fprintd-verify`: **verify-match**
- `sudo` PAM: prompts for fingerprint and reaches the standard `pam_fprintd` path

## Maintainer goals / Roadmap

- Prepare an upstreamable patch series by splitting the current export into focused changes for hardware ID enablement, persistent pairing data, template fallback behavior, and local test hooks.
- Expand the validation matrix for enroll, verify, list-prints, `sudo` PAM, duplicate-enrollment handling, fallback-template matching, and negative cases such as missing persistent data.
- Keep the build and smoke-test workflow reproducible against fresh upstream `libfprint` and `fprintd` checkouts instead of relying on local state.
- Maintain a public data-hygiene checklist so fingerprint templates, pairing blobs, `/var/lib/fprint`, per-user runtime state, and raw biometric logs never enter the repository.
- Triage external reports as structured issues with hardware model, sensor firmware, distro, kernel, libfprint/fprintd refs, reproduction steps, expected result, and actual result.

## Suggested layout

The helper scripts assume a workspace shaped like this:

```text
repo-root/
  patches/
  scripts/
  upstream/
    libfprint/
    fprintd/
```

The patches are intended to apply against upstream `libfprint` and `fprintd` checkouts in `upstream/`.
The wrapper scripts can be adapted for other layouts by changing `PROJECT_ROOT` handling.

## Recommended next steps

1. apply the patches to fresh upstream `libfprint` / `fprintd` trees
2. use the build helpers in `scripts/`
3. validate with the local wrapper flow first
4. follow `docs/REPRODUCTION.md` to record the exact refs, commands, and results
5. only then activate the build system-wide with the notes in `docs/system-activation-notes.md`

## Safety note

If you publish this repository further, keep it private or re-audit the contents first. The code is safe to share,
but runtime storage and logs from a real device session are intentionally excluded and should stay excluded.
