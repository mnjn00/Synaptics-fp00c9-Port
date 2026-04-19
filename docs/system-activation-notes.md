# System activation notes

These notes summarize the working activation path that was validated after local testing.

## Installed runtime pieces

- `/usr/local/lib/libfprint-2.so*`
- `/usr/local/libexec/fprintd`
- `/usr/local/lib/security/pam_fprintd.so`
- `/usr/local/etc/fprintd.conf`

## DBus/systemd wiring

Use a DBus service file and a `fprintd.service` systemd override so the system daemon resolves to the
custom `/usr/local/libexec/fprintd` build instead of the distro package.

## PAM wiring that was validated

- `system-auth` — `pam_fprintd.so` before `pam_unix.so`
- `sudo` — explicit `pam_fprintd.so max-tries=3 timeout=30`, then include `system-auth`
- login/display-manager PAM files — `pam_fprintd.so` inserted before password auth

## Activation advice

1. keep backups of every PAM file before editing
2. validate `fprintd-list` and `fprintd-verify` as the target user before touching PAM
3. validate `sudo -K && sudo true` before enabling login-screen fingerprint auth
4. only after that, wire the display manager / lock screen

## Data hygiene

Do **not** copy any real `/var/lib/fprint` data, pairing state, or per-user runtime state into a public repo.
