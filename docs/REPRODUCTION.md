# Reproduction guide

This guide is for maintainers who want to reproduce the `06cb:00c9` standard-stack bring-up from a clean workspace.
It intentionally avoids copying any live biometric runtime data into the repository.

## Scope

Use this guide to validate that the patch set can be applied, built, and tested against fresh upstream checkouts of:

- `libfprint`
- `fprintd`

The expected end state is a local build that can enroll and verify a Synaptics `06cb:00c9` match-in-sensor device through the standard `libfprint` / `fprintd` path before any system-wide PAM activation.

## Workspace layout

Create a workspace shaped like this:

```text
repo-root/
  patches/
  scripts/
  upstream/
    libfprint/
    fprintd/
```

Record the exact upstream commits used for each run:

```bash
git -C upstream/libfprint rev-parse HEAD
git -C upstream/fprintd rev-parse HEAD
```

## Apply patches

Apply the patch set to fresh upstream trees:

```bash
git -C upstream/libfprint apply ../../patches/libfprint-standard-stack.patch
git -C upstream/fprintd apply ../../patches/fprintd-standard-stack.patch
```

If a patch does not apply cleanly, record:

- upstream project name
- upstream commit SHA
- failed file and hunk
- local distro, kernel, and package versions
- whether the failure is a simple context drift or a semantic conflict

## Local build checklist

Before touching system services or PAM files, build and stage the local copies only.
Use the helper scripts in `scripts/` where possible, or keep equivalent commands in a local note.

Minimum information to capture for every reproduction attempt:

```text
Date:
Host model:
Sensor USB ID:
Distro:
Kernel:
libfprint upstream SHA:
fprintd upstream SHA:
Patch-set commit SHA:
Build result:
Local enroll result:
Local verify result:
System activation attempted: yes/no
```

## Validation order

Follow this order so that failures stay narrow and recoverable:

1. Confirm the device appears as Synaptics `06cb:00c9`.
2. Build patched `libfprint` and confirm the sensor is detected by the local tooling.
3. Run local enrollment before installing anything system-wide.
4. Run local verification against the enrolled local print.
5. Run `fprintd-list` and `fprintd-verify` only after the local flow succeeds.
6. Activate PAM integration only after `fprintd` validation succeeds.
7. Validate `sudo -K && sudo true` before enabling display-manager or lock-screen fingerprint authentication.

## Expected results

A successful reproduction should reach:

- local standard-stack enrollment completes
- local standard-stack verification matches
- `fprintd-list` can see the enrolled finger after controlled activation
- `fprintd-verify` returns a verify-match result
- `sudo` reaches the `pam_fprintd` path without breaking password fallback

## Negative cases to test

When possible, also record these cases:

- missing persistent pairing data
- no enrolled prints
- duplicate enrollment attempt
- template-only enrollment fallback
- verification with the wrong finger
- `sudo` timeout or retry path
- fprintd restart after enrollment

## Data hygiene checklist

Before committing any reproduction artifact, confirm that the repository does **not** contain:

- enrolled fingerprint templates
- pairing blobs or persistent sensor state
- `~/.local/state/fp00c9` runtime data
- `/var/lib/fprint` contents
- raw enrollment or verification logs containing live biometric session data
- local build directories such as `builddir`, `stage`, `fprintd-stage`, or `.venv`

Prefer summarized results over raw logs. If a raw log is needed for debugging, redact user names, hardware serials, template identifiers, and biometric-session payloads before sharing.

## Issue report template

External reports should include:

```text
Hardware model:
Sensor USB ID:
Distro and version:
Kernel version:
libfprint commit:
fprintd commit:
Patch-set commit:
Command that failed:
Expected result:
Actual result:
Sanitized logs or screenshots:
```

Do not attach fingerprint templates, pairing blobs, `/var/lib/fprint`, or per-user runtime directories.
