# Bring-up helper scripts

The scripts below are the sanitized local helpers used during port bring-up.

## `setup-local-build-tools.sh`

```sh
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"


SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
VENV_DIR="${VENV_DIR:-$HOME/.venvs/libfprint-build}"
BIN_DIR="$VENV_DIR/bin"
GLIB_GIO_DIR="$VENV_DIR/share/glib-gio"

python3 -m venv "$VENV_DIR"
"$BIN_DIR/pip" install meson ninja

curl -L --fail --silent \
  https://gitlab.gnome.org/GNOME/glib/-/raw/main/gobject/glib-mkenums.in |
  sed '1s|#!@PYTHON@|#!/usr/bin/env python3|' > "$BIN_DIR/glib-mkenums"
chmod +x "$BIN_DIR/glib-mkenums"

if [[ ! -d "$GLIB_GIO_DIR/codegen" ]]; then
  TMP_CLONE="$(mktemp -d)"
  trap 'rm -rf "$TMP_CLONE"' EXIT
  git clone --depth=1 --filter=blob:none --sparse https://gitlab.gnome.org/GNOME/glib.git "$TMP_CLONE"
  git -C "$TMP_CLONE" sparse-checkout set gio/gdbus-2.0/codegen
  mkdir -p "$GLIB_GIO_DIR/codegen"
  cp -a "$TMP_CLONE/gio/gdbus-2.0/codegen/." "$GLIB_GIO_DIR/codegen/"
fi

cat > "$GLIB_GIO_DIR/codegen/config.py" <<'PY'
VERSION = '2.88.0'
MAJOR_VERSION = 2
MINOR_VERSION = 88
PY

cat > "$BIN_DIR/gdbus-codegen" <<EOF
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

export PYTHONPATH="$GLIB_GIO_DIR:\${PYTHONPATH:-}"
exec /usr/bin/env python3 "$GLIB_GIO_DIR/codegen/gdbus-codegen.in" "\$@"
EOF
chmod +x "$BIN_DIR/gdbus-codegen"

cat > "$BIN_DIR/pkg-config-wrapper" <<EOF
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

if [[ "\$#" -eq 2 && "\$1" == "--variable=glib_mkenums" && "\$2" == "glib-2.0" ]]; then
  printf '%s\n' "$BIN_DIR/glib-mkenums"
  exit 0
fi
if [[ "\$#" -eq 2 && "\$1" == "--variable=gdbus_codegen" && "\$2" == "gio-2.0" ]]; then
  printf '%s\n' "$BIN_DIR/gdbus-codegen"
  exit 0
fi
exec /usr/bin/pkg-config "\$@"
EOF
chmod +x "$BIN_DIR/pkg-config-wrapper"

cat <<EOF
Local libfprint build tools are ready.
  VENV_DIR=$VENV_DIR
  MESON=$BIN_DIR/meson
  NINJA=$BIN_DIR/ninja
  GDBUS_CODEGEN=$BIN_DIR/gdbus-codegen
  PKG_CONFIG=$BIN_DIR/pkg-config-wrapper
EOF
```

## `build-local-libfprint.sh`

```sh
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"


SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
SOURCE_DIR="$SCRIPT_DIR/libfprint"
BUILD_DIR="${BUILD_DIR:-$SOURCE_DIR/builddir}"
VENV_DIR="${VENV_DIR:-$HOME/.venvs/libfprint-build}"
BIN_DIR="$VENV_DIR/bin"
STAGE_DIR="${STAGE_DIR:-$SCRIPT_DIR/stage}"

if [[ ! -x "$BIN_DIR/meson" || ! -x "$BIN_DIR/ninja" || ! -x "$BIN_DIR/pkg-config-wrapper" ]]; then
  "$SCRIPT_DIR/setup-local-build-tools.sh"
fi

export PATH="$BIN_DIR:$PATH"
export PKG_CONFIG="$BIN_DIR/pkg-config-wrapper"

rm -rf "$BUILD_DIR"
"$BIN_DIR/meson" setup "$BUILD_DIR" "$SOURCE_DIR" \
  -Dintrospection=false \
  -Ddoc=false \
  -Dinstalled-tests=false
"$BIN_DIR/meson" compile -C "$BUILD_DIR"
"$BIN_DIR/ninja" -C "$BUILD_DIR" sync-udev-hwdb
rm -rf "$STAGE_DIR"
DESTDIR="$STAGE_DIR" "$BIN_DIR/meson" install -C "$BUILD_DIR"

cat <<EOF
libfprint build completed.
  SOURCE_DIR=$SOURCE_DIR
  BUILD_DIR=$BUILD_DIR
  STAGE_DIR=$STAGE_DIR
  Example supported-device check:
    $BUILD_DIR/libfprint/fprint-list-supported-devices | grep -i '06cb:00c9'
EOF
```

## `build-local-fprintd.sh`

```sh
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
FPRINTD_DIR="${FPRINTD_DIR:-${PROJECT_ROOT}/fprintd}"
VENV_DIR="${VENV_DIR:-$HOME/.venvs/libfprint-build}"
BIN_DIR="$VENV_DIR/bin"
LIBFPRINT_PKGCONFIG_DIR="${LIBFPRINT_PKGCONFIG_DIR:-${PROJECT_ROOT}/libfprint/local-pkgconfig}"
STAGE_DIR="${STAGE_DIR:-$SCRIPT_DIR/fprintd-stage}"

if [[ ! -d "$FPRINTD_DIR/.git" ]]; then
  git clone --depth=1 https://gitlab.freedesktop.org/libfprint/fprintd.git "$FPRINTD_DIR"
fi

if ! git -C "$FPRINTD_DIR" diff --quiet; then
  :
elif git -C "$FPRINTD_DIR" apply --check "$SCRIPT_DIR/fprintd-load-store-persistent-data-from-device.patch" >/dev/null 2>&1; then
  git -C "$FPRINTD_DIR" apply "$SCRIPT_DIR/fprintd-load-store-persistent-data-from-device.patch"
fi

FPRINTD_DIR="$FPRINTD_DIR" python3 - <<'PY'
import os
from pathlib import Path
path = Path(os.environ["FPRINTD_DIR"]) / "src/device.c"
text = path.read_text()
old = """        case FP_DEVICE_RETRY_TOO_FAST:\n          return \"verify-too-fast\";\n"""
new = """#ifdef FP_DEVICE_RETRY_TOO_FAST\n        case FP_DEVICE_RETRY_TOO_FAST:\n          return \"verify-too-fast\";\n#endif\n"""
text = text.replace(old, new)
old = """        case FP_DEVICE_RETRY_TOO_FAST:\n          return \"enroll-too-fast\";\n"""
new = """#ifdef FP_DEVICE_RETRY_TOO_FAST\n        case FP_DEVICE_RETRY_TOO_FAST:\n          return \"enroll-too-fast\";\n#endif\n"""
text = text.replace(old, new)
path.write_text(text)
PY

export PATH="$BIN_DIR:$PATH"
export PKG_CONFIG="$BIN_DIR/pkg-config-wrapper"
export PKG_CONFIG_PATH="$LIBFPRINT_PKGCONFIG_DIR"

rm -rf "$FPRINTD_DIR/builddir"
"$BIN_DIR/meson" setup "$FPRINTD_DIR/builddir" "$FPRINTD_DIR" -Dgtk_doc=false -Dman=false
"$BIN_DIR/meson" compile -C "$FPRINTD_DIR/builddir"
rm -rf "$STAGE_DIR"
DESTDIR="$STAGE_DIR" "$BIN_DIR/meson" install -C "$FPRINTD_DIR/builddir"

cat <<EOF
fprintd build completed against the staged local libfprint.
  FPRINTD_DIR=$FPRINTD_DIR
  BUILD_DIR=$FPRINTD_DIR/builddir
  STAGE_DIR=$STAGE_DIR
  Local daemon binary: $FPRINTD_DIR/builddir/src/fprintd
EOF
```

## `run-local-libfprint-example.sh`

```sh
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"


SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd)"
BUILD_DIR="${BUILD_DIR:-$SCRIPT_DIR/libfprint/builddir}"
STAGE_LIB_DIR="${STAGE_LIB_DIR:-$SCRIPT_DIR/stage/usr/local/lib}"

if [[ $# -lt 1 ]]; then
  echo "usage: $0 <manage-prints|enroll|verify|identify|img-capture> [args...]"
  exit 2
fi

example="$1"
shift

if [[ -z "${FP_EXAMPLE_STORAGE_FILE:-}" ]]; then
  state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/fp00c9"
  mkdir -p "$state_dir"
  export FP_EXAMPLE_STORAGE_FILE="$state_dir/libfprint-example-storage.variant"
fi

exec env \
  LD_LIBRARY_PATH="$STAGE_LIB_DIR${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}" \
  "$BUILD_DIR/examples/$example" "$@"
```

## `fp00c9-warmup.py`

```py
#!/usr/bin/python3
import os, sys, time, struct, usb.core, usb.util
RESET='${FP00C9_RESET_USB:-/usr/local/bin/fp00c9-reset-usb}'
VID=0x06cb
PID=0x00c9

def reset_usb():
    if os.geteuid() == 0:
        os.system(RESET + ' >/dev/null 2>&1')

def try_probe():
    dev = usb.core.find(idVendor=VID, idProduct=PID)
    if dev is None:
        return False
    try:
        dev.set_configuration()
        if dev.is_kernel_driver_active(0):
            try: dev.detach_kernel_driver(0)
            except Exception: pass
        usb.util.claim_interface(dev, 0)
        intf = dev.get_active_configuration()[(0,0)]
        cmd_ep = intf[0]
        resp_ep = intf[1]
        cmd_ep.write(b'\x01', 2000)
        data = bytes(resp_ep.read(0x26, 2000))
        return len(data) >= 2 and struct.unpack('<H', data[:2])[0] == 0
    except Exception:
        return False
    finally:
        try: usb.util.release_interface(dev, 0)
        except Exception: pass
        try: dev.reset()
        except Exception: pass

for _ in range(5):
    reset_usb()
    time.sleep(1)
    if try_probe():
        sys.exit(0)
    time.sleep(1)
sys.exit(1)
```

## `fp00c9-standard-fprintd-logwatch`

```sh
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"


mode="${1:-}"
logfile="${2:-}"

if [[ -z "$mode" || -z "$logfile" ]]; then
  echo "usage: $0 <mode> <daemon-log>" >&2
  exit 2
fi

tail -n0 -F "$logfile" | while IFS= read -r line; do
  : "${finger_seen:=0}"
  : "${enroll_state:=}"
  : "${identify_state:=}"

  case "$line" in
    *"Enroll entering state "*)
      enroll_state="${line##*Enroll entering state }"
      ;;
    *"Identify/Verify entering state "*)
      identify_state="${line##*Identify/Verify entering state }"
      ;;
  esac

  case "$line" in
    *"Need to pair sensor"*)
      echo "[fp00c9] Sensor needs pairing for this session; continuing automatically."
      ;;
    *"Loading pairing data form persistent storage:"*)
      echo "[fp00c9] Reusing stored pairing data."
      ;;
    *"Waiting for events..."*)
      case "$mode" in
        enroll)
          if [[ "$enroll_state" == "2" ]]; then
            echo "[fp00c9] Keep your finger OFF the sensor for this stage."
          elif [[ "$finger_seen" -eq 1 ]]; then
            echo "[fp00c9] Still processing this scan. Keep holding your finger still for up to 20 seconds."
          else
            echo "[fp00c9] Scan window is open NOW. Place your right index finger."
          fi
          ;;
        verify)
          if [[ "$identify_state" == "1" ]]; then
            echo "[fp00c9] Keep your finger OFF the sensor for this stage."
          elif [[ "$finger_seen" -eq 1 ]]; then
            echo "[fp00c9] Still processing this verify scan. Keep holding your finger still for up to 20 seconds."
          else
            echo "[fp00c9] Verify window is open NOW. Place an enrolled finger."
          fi
          ;;
        enroll-verify)
          if [[ "$enroll_state" == "2" || "$identify_state" == "1" ]]; then
            echo "[fp00c9] Keep your finger OFF the sensor for this stage."
          elif [[ "$finger_seen" -eq 1 ]]; then
            echo "[fp00c9] Still processing this scan. Keep holding your finger still for up to 20 seconds."
          else
            echo "[fp00c9] Scan window is open NOW. Place your finger and hold it still."
          fi
          ;;
        *)
          echo "[fp00c9] Sensor is waiting for a finger now."
          ;;
      esac
      ;;
    *"FINGER_DOWN"*)
      echo "[fp00c9] Finger detected. Keep holding still until told to lift; this stage may take 15-20 seconds."
      finger_seen=1
      ;;
    *"FINGER_UP"*)
      echo "[fp00c9] Finger is off the sensor."
      finger_seen=0
      ;;
    *"FRAME_READY"*)
      echo "[fp00c9] Frame ready."
      ;;
    *"enroll-stage-passed"*)
      echo "[fp00c9] One enrollment stage passed. Lift your finger and wait for the next scan window."
      finger_seen=0
      ;;
    *"enroll_stage_cb: completed_stages="*)
      stages="${line##*completed_stages=}"
      echo "[fp00c9] Enrollment progress: $stages"
      ;;
    *"enroll-remove-and-retry"*)
      echo "[fp00c9] Remove your finger and try again."
      finger_seen=0
      ;;
    *"enroll-retry-scan"*)
      echo "[fp00c9] Retry the scan."
      ;;
    *"enroll-completed"*)
      echo "[fp00c9] Enrollment completed."
      finger_seen=0
      ;;
    *"verify-match"*)
      echo "[fp00c9] Verify matched."
      finger_seen=0
      ;;
    *"verify-no-match"*)
      echo "[fp00c9] Verify did not match."
      finger_seen=0
      ;;
    *"verify-remove-and-retry"*)
      echo "[fp00c9] Remove your finger and retry verify."
      finger_seen=0
      ;;
    *"verify-retry-scan"*)
      echo "[fp00c9] Retry the verify scan."
      finger_seen=0
      ;;
  esac
done
```

## `fp00c9-standard-fprintd-list-once`

```sh
#!/usr/bin/env bash
set -euo pipefail

state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/fp00c9"
mkdir -p "$state_dir"
storage_dir="$state_dir/fprintd-storage"
mkdir -p "$storage_dir"
run_dir="$state_dir/fprintd-runtime"
mkdir -p "$run_dir"

daemon_log="$state_dir/standard-fprintd-daemon.log"
client_log="$state_dir/standard-fprintd-list.log"

cat <<EOF
[fp00c9] local standard fprintd list
- session bus sandbox: enabled
- local daemon: ${PROJECT_ROOT}/fprintd/builddir/src/fprintd
- storage dir: $storage_dir
- this uses the patched local standard-stack fprintd/libfprint, not the system daemon

EOF

timeout 15 fp00c9-warmup >/dev/null 2>&1 || true

dbus-run-session -- bash -lc "
  set -euo pipefail
  export LD_LIBRARY_PATH=${PROJECT_ROOT}/libfprint/stage/usr/local/lib
  export FPRINTD_USE_SESSION_BUS=1
  export FPRINTD_SESSION_TEST_TRUST=1
  export FPRINTD_STORAGE_DIR='$storage_dir'
  export FPRINTD_FORCE_USER=\"${FP00C9_FORCE_USER:-$(id -un)}\"
  export HOME='$run_dir'
  export G_MESSAGES_DEBUG=all
  ${PROJECT_ROOT}/fprintd/builddir/src/fprintd >'$daemon_log' 2>&1 &
  pid=\$!
  sleep 3
  ${PROJECT_ROOT}/fprintd/builddir/utils/fprintd-list \"\$(whoami)\" 2>&1 | tee '$client_log'
  kill \$pid || true
  wait \$pid || true
"
```

## `fp00c9-standard-fprintd-enroll-once`

```sh
#!/usr/bin/env bash
set -euo pipefail

state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/fp00c9"
mkdir -p "$state_dir"
storage_dir="$state_dir/fprintd-storage"
mkdir -p "$storage_dir"
run_dir="$state_dir/fprintd-runtime"
mkdir -p "$run_dir"

daemon_log="$state_dir/standard-fprintd-daemon.log"
client_log="$state_dir/standard-fprintd-enroll.log"
status_log="$state_dir/standard-fprintd-enroll.status"

cat <<EOF
[fp00c9] local standard fprintd enroll
- target finger: right-index-finger
- session bus sandbox: enabled
- local daemon: ${PROJECT_ROOT}/fprintd/builddir/src/fprintd
- storage dir: $storage_dir
- this uses the patched local standard-stack fprintd/libfprint, not the system daemon
- when it prints 'Enrolling right-index-finger finger.' place your right index finger on the sensor.
- keep the finger still and follow repeated scans until enrollment completes successfully.

EOF

timeout 15 fp00c9-warmup >/dev/null 2>&1 || true

run_once() {
  local attempt="$1"
  local attempt_daemon_log="${daemon_log}.attempt${attempt}"
  local attempt_client_log="${client_log}.attempt${attempt}"
  echo "[fp00c9] standard fprintd attempt $attempt/3"
  dbus-run-session -- bash -lc "
    set -euo pipefail
    export LD_LIBRARY_PATH=${PROJECT_ROOT}/libfprint/stage/usr/local/lib
    export FPRINTD_USE_SESSION_BUS=1
    export FPRINTD_SESSION_TEST_TRUST=1
    export FPRINTD_STORAGE_DIR='$storage_dir'
    export FPRINTD_SKIP_DUPLICATE_CHECK=1
    export FPRINTD_FORCE_USER=\"${FP00C9_FORCE_USER:-$(id -un)}\"
    export HOME='$run_dir'
    export G_MESSAGES_DEBUG=all
    : >'$attempt_daemon_log'
    : >'$status_log'
    ${PROJECT_ROOT}/fprintd/builddir/src/fprintd >'$attempt_daemon_log' 2>&1 &
    pid=\$!
    fp00c9-standard-fprintd-logwatch enroll '$attempt_daemon_log' &
    tail_pid=\$!
    sleep 3
    stdbuf -oL -eL timeout 300 ${PROJECT_ROOT}/fprintd/builddir/utils/fprintd-enroll -f right-index-finger \"\$(whoami)\" 2>&1 | tee '$attempt_client_log' || true
    cp '$attempt_daemon_log' '$daemon_log' || true
    cp '$attempt_client_log' '$client_log' || true
    if grep -q 'endpoint stalled or request not supported' '$attempt_client_log'; then
      echo endpoint_stall >'$status_log'
    elif grep -q 'Enroll result: enroll-completed' '$attempt_client_log'; then
      echo enroll_completed >'$status_log'
    else
      echo enroll_incomplete >'$status_log'
    fi
    kill \$tail_pid || true
    kill \$pid || true
    wait \$tail_pid || true
    wait \$pid || true
  "
}

for attempt in 1 2 3; do
  run_once "$attempt"
  if [[ -f "$status_log" ]]; then
    status="$(cat "$status_log")"
    if [[ "$status" == "endpoint_stall" && "$attempt" -lt 3 ]]; then
      echo "[fp00c9] endpoint stalled; retrying..."
      sleep 2
      continue
    fi
  fi
  break
done

if [[ -f "$client_log" || -f "$daemon_log" ]]; then
  if grep -q 'Enroll result: enroll-completed' "$client_log" 2>/dev/null; then
    echo "[fp00c9] Final status: enrollment completed."
  elif grep -q 'enroll-stage-passed' "$daemon_log" 2>/dev/null; then
    echo "[fp00c9] Final status: at least one enrollment stage passed, but enrollment did not complete in this run."
  elif grep -q 'FRAME_READY' "$daemon_log" 2>/dev/null; then
    echo "[fp00c9] Final status: frame-ready was seen, but no completed enrollment stage was reported before the run ended."
  elif grep -q 'FINGER_DOWN' "$daemon_log" 2>/dev/null; then
    echo "[fp00c9] Final status: finger was detected, but the sensor never reported FRAME_READY or a completed enrollment stage before this run ended."
  else
    echo "[fp00c9] Final status: no usable enrollment progress was observed in this run."
  fi
fi
```

## `fp00c9-standard-fprintd-verify-once`

```sh
#!/usr/bin/env bash
set -euo pipefail

state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/fp00c9"
mkdir -p "$state_dir"
storage_dir="$state_dir/fprintd-storage"
mkdir -p "$storage_dir"
run_dir="$state_dir/fprintd-runtime"
mkdir -p "$run_dir"

daemon_log="$state_dir/standard-fprintd-daemon.log"
client_log="$state_dir/standard-fprintd-verify.log"
status_log="$state_dir/standard-fprintd-verify.status"

cat <<EOF
[fp00c9] local standard fprintd verify
- target mode: any enrolled finger
- session bus sandbox: enabled
- local daemon: ${PROJECT_ROOT}/fprintd/builddir/src/fprintd
- storage dir: $storage_dir
- this uses the patched local standard-stack fprintd/libfprint, not the system daemon
- after the device is claimed, place an enrolled finger on the sensor when prompted by the device
- expected success line is typically:
  verify-match

EOF

timeout 15 fp00c9-warmup >/dev/null 2>&1 || true

run_once() {
  local attempt="$1"
  echo "[fp00c9] standard fprintd attempt $attempt/3"
  dbus-run-session -- bash -lc "
    set -euo pipefail
    export LD_LIBRARY_PATH=${PROJECT_ROOT}/libfprint/stage/usr/local/lib
    export FPRINTD_USE_SESSION_BUS=1
    export FPRINTD_SESSION_TEST_TRUST=1
    export FPRINTD_STORAGE_DIR='$storage_dir'
    export FPRINTD_DEVICE_LIST_FALLBACK=1
    export FPRINTD_SKIP_LIST_ENROLLED=1
    export FPRINTD_FORCE_USER=\"${FP00C9_FORCE_USER:-$(id -un)}\"
    export HOME='$run_dir'
    export G_MESSAGES_DEBUG=all
    : >'$daemon_log'
    : >'$status_log'
    ${PROJECT_ROOT}/fprintd/builddir/src/fprintd >'$daemon_log' 2>&1 &
    pid=\$!
    fp00c9-standard-fprintd-logwatch verify '$daemon_log' &
    tail_pid=\$!
    sleep 3
    stdbuf -oL -eL timeout 180 ${PROJECT_ROOT}/fprintd/builddir/utils/fprintd-verify \"\$(whoami)\" 2>&1 | tee '$client_log' || true
    if grep -q 'endpoint stalled or request not supported' '$client_log'; then
      echo endpoint_stall >'$status_log'
    elif grep -q 'Verify result: verify-match' '$client_log'; then
      echo verify_match >'$status_log'
    else
      echo verify_incomplete >'$status_log'
    fi
    kill \$tail_pid || true
    kill \$pid || true
    wait \$tail_pid || true
    wait \$pid || true
  "
}

for attempt in 1 2 3; do
  run_once "$attempt"
  if [[ -f "$status_log" ]]; then
    status="$(cat "$status_log")"
    if [[ "$status" == "endpoint_stall" && "$attempt" -lt 3 ]]; then
      echo "[fp00c9] endpoint stalled; retrying..."
      sleep 2
      continue
    fi
  fi
  break
done
```

## `fp00c9-standard-fprintd-enroll-verify-once`

```sh
#!/usr/bin/env bash
set -euo pipefail

state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/fp00c9"
mkdir -p "$state_dir"
storage_dir="$state_dir/fprintd-storage"
mkdir -p "$storage_dir"
run_dir="$state_dir/fprintd-runtime"
mkdir -p "$run_dir"

daemon_log="$state_dir/standard-fprintd-daemon.log"
enroll_log="$state_dir/standard-fprintd-enroll.log"
verify_log="$state_dir/standard-fprintd-verify.log"
status_log="$state_dir/standard-fprintd-enroll-verify.status"

cat <<EOF
[fp00c9] local standard fprintd enroll+verify
- target finger: right-index-finger
- session bus sandbox: enabled
- local daemon: ${PROJECT_ROOT}/fprintd/builddir/src/fprintd
- storage dir: $storage_dir
- this uses the patched local standard-stack fprintd/libfprint, not the system daemon
- keep your finger OFF until:
  Enrolling right-index-finger finger.
- then place your right index finger and keep following scans until enrollment completes successfully.
- if enrollment completes, this command will immediately launch verify next.

EOF

timeout 15 fp00c9-warmup >/dev/null 2>&1 || true

run_once() {
  local attempt="$1"
  local attempt_daemon_log="${daemon_log}.attempt${attempt}"
  local attempt_enroll_log="${enroll_log}.attempt${attempt}"
  local attempt_verify_log="${verify_log}.attempt${attempt}"
  echo "[fp00c9] standard fprintd attempt $attempt/3"
  dbus-run-session -- bash -lc "
    set -euo pipefail
    export LD_LIBRARY_PATH=${PROJECT_ROOT}/libfprint/stage/usr/local/lib
    export FPRINTD_USE_SESSION_BUS=1
    export FPRINTD_SESSION_TEST_TRUST=1
    export FPRINTD_STORAGE_DIR='$storage_dir'
    export FPRINTD_DEVICE_LIST_FALLBACK=1
    export FPRINTD_SKIP_LIST_ENROLLED=1
    export FPRINTD_SKIP_DUPLICATE_CHECK=1
    export FPRINTD_FORCE_USER=\"${FP00C9_FORCE_USER:-$(id -un)}\"
    export HOME='$run_dir'
    export G_MESSAGES_DEBUG=all
    : >'$attempt_daemon_log'
    : >'$status_log'
    ${PROJECT_ROOT}/fprintd/builddir/src/fprintd >'$attempt_daemon_log' 2>&1 &
    pid=\$!
    fp00c9-standard-fprintd-logwatch enroll-verify '$attempt_daemon_log' &
    tail_pid=\$!
    sleep 3
    stdbuf -oL -eL timeout 300 ${PROJECT_ROOT}/fprintd/builddir/utils/fprintd-enroll -f right-index-finger \"\$(whoami)\" 2>&1 | tee '$attempt_enroll_log' || true
    cp '$attempt_daemon_log' '$daemon_log' || true
    cp '$attempt_enroll_log' '$enroll_log' || true
    if grep -q 'Enroll result: enroll-completed' '$attempt_enroll_log'; then
      echo enroll_completed >'$status_log'
      echo
      echo '[fp00c9] enrollment completed; starting verify...'
      stdbuf -oL -eL timeout 180 ${PROJECT_ROOT}/fprintd/builddir/utils/fprintd-verify \"\$(whoami)\" 2>&1 | tee '$attempt_verify_log' || true
      cp '$attempt_verify_log' '$verify_log' || true
    elif grep -q 'endpoint stalled or request not supported' '$attempt_enroll_log'; then
      echo endpoint_stall >'$status_log'
    else
      echo enroll_incomplete >'$status_log'
    fi
    kill \$tail_pid || true
    kill \$pid || true
    wait \$tail_pid || true
    wait \$pid || true
  "
}

for attempt in 1 2 3; do
  run_once "$attempt"
  if [[ -f "$status_log" ]]; then
    status="$(cat "$status_log")"
    if [[ "$status" == "endpoint_stall" && "$attempt" -lt 3 ]]; then
      echo "[fp00c9] endpoint stalled; retrying..."
      sleep 2
      continue
    fi
  fi
  break
done

if [[ -f "$verify_log" ]] && grep -q 'Verify result: verify-match' "$verify_log" 2>/dev/null; then
  echo "[fp00c9] Final status: verify matched."
elif [[ -f "$enroll_log" ]] && grep -q 'Enroll result: enroll-completed' "$enroll_log" 2>/dev/null; then
  echo "[fp00c9] Final status: enrollment completed, but verify did not report a match in this run."
elif grep -q 'enroll-stage-passed' "$daemon_log" 2>/dev/null; then
  echo "[fp00c9] Final status: at least one enrollment stage passed, but enrollment did not complete in this run."
elif grep -q 'FRAME_READY' "$daemon_log" 2>/dev/null; then
  echo "[fp00c9] Final status: frame-ready was seen, but no completed enrollment stage was reported before the run ended."
elif grep -q 'FINGER_DOWN' "$daemon_log" 2>/dev/null; then
  echo "[fp00c9] Final status: finger was detected, but the sensor never reported FRAME_READY or a completed enrollment stage before the run ended."
else
  echo "[fp00c9] Final status: no useful enrollment/verify progress was observed in this run."
fi
```

## `fp00c9-standard-fprintd-quick`

```sh
#!/usr/bin/env bash
set -euo pipefail

PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"


state_dir="${XDG_STATE_HOME:-$HOME/.local/state}/fp00c9"

latest_log() {
  local base="$1"
  local path="$state_dir/$base"
  if [[ -f "$path" ]]; then
    printf '%s\n' "$path"
    return 0
  fi
  local latest
  latest="$(ls -1t "$path".attempt* 2>/dev/null | head -n1 || true)"
  if [[ -n "$latest" ]]; then
    printf '%s\n' "$latest"
  fi
}

echo "=== fp00c9 local fprintd summary ==="
echo "state dir: $state_dir"

enroll_log="$(latest_log standard-fprintd-enroll.log)"
verify_log="$(latest_log standard-fprintd-verify.log)"
daemon_log="$(latest_log standard-fprintd-daemon.log)"

if [[ -f "$daemon_log" || -f "$enroll_log" || -f "$verify_log" ]]; then
  echo
  echo "--- interpreted status ---"
  if grep -q 'Enroll result: enroll-completed' "$enroll_log" 2>/dev/null; then
    echo "enroll: completed"
  elif grep -q 'enroll-stage-passed' "$daemon_log" 2>/dev/null; then
    echo "enroll: at least one stage passed"
  elif grep -q 'FRAME_READY' "$daemon_log" 2>/dev/null; then
    echo "enroll: frame-ready seen but no completed stage observed yet"
  elif grep -q 'FINGER_DOWN' "$daemon_log" 2>/dev/null; then
    echo "enroll: finger detected but no completed stage observed yet"
  elif grep -q 'Enrolling right-index-finger finger\.' "$enroll_log" 2>/dev/null; then
    echo "enroll: started but no finger-detect evidence yet"
  else
    echo "enroll: no recent evidence"
  fi

  if grep -q 'Verify result: verify-match' "$verify_log" 2>/dev/null; then
    echo "verify: matched"
  elif grep -q 'Verify result: verify-no-match' "$verify_log" 2>/dev/null; then
    echo "verify: explicit no-match"
  elif grep -q 'Verify started!' "$verify_log" 2>/dev/null; then
    echo "verify: started but no final result observed"
  else
    echo "verify: no recent evidence"
  fi
fi

for log in standard-fprintd-list.log standard-fprintd-enroll.log standard-fprintd-verify.log standard-fprintd-daemon.log; do
  path="$(latest_log "$log")"
  if [[ -n "$path" && -f "$path" ]]; then
    echo
    echo "--- $(basename "$path") ---"
    grep -E 'Using device|found 1 devices|has no fingers enrolled|Enrolling right-index-finger finger\\.|Verify started!|Verify result:|Verifying:|verify-match|verify-no-match|ListEnrolledFingers failed|Need to pair sensor|Loading pairing data form persistent storage|Failed to load persistent data|Enroll entering state|Waiting for events|Frame acquire|FINGER_DOWN|FRAME_READY|Added fallback print' "$path" || true
  fi
done
```
