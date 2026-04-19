# fp00c9-standard-stack-port

Patch set, helper scripts, and activation notes for bringing the Synaptics `06cb:00c9`
match-in-sensor device onto the standard Linux fingerprint stack (`libfprint` + `fprintd` + PAM).

## What this repo contains

- `patches/libfprint-standard-stack.patch` — libfprint changes that enabled and stabilized the sensor.
- `patches/fprintd-standard-stack.patch` — fprintd changes for local testing, pairing persistence, and template fallback.
- `patches/pydrv-analysis-fixes.patch` — Python-driver fixes that informed the libfprint port.
- `scripts/` — build helpers plus local smoke-test wrappers used during bring-up.
- `docs/` — notes for local validation and system activation.

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
4. only then activate the build system-wide with the notes in `docs/system-activation-notes.md`

## Safety note

If you publish this repository further, keep it private or re-audit the contents first. The code is safe to share,
but runtime storage and logs from a real device session are intentionally excluded and should stay excluded.
