# Reproduction outline

## 1. Clone upstreams

- `libfprint`
- `fprintd`

Place them under `upstream/` or adapt the helper scripts.

## 2. Apply patches

- `patches/libfprint-standard-stack.patch`
- `patches/fprintd-standard-stack.patch`
- optionally inspect `patches/pydrv-analysis-fixes.patch`

## 3. Build

Use:

- `scripts/setup-local-build-tools.sh`
- `scripts/build-local-libfprint.sh`
- `scripts/build-local-fprintd.sh`

## 4. Local validation

Recommended order:

- local list
- local enroll
- local verify
- quick summary

The exported wrapper scripts are examples of the flow that was validated during bring-up.
