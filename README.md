## iwctlWithSystemd

Use systemd-networkd with iwd for a lightweight, minimal networking solution without bloated network managers.

1. Install the iwd package:

   ```bash
   pacman -S iwd
   ```

2. Enable the iwd service:

   ```bash
   systemctl enable iwd
   ```

3. Make sure systemd is installed:

   ```bash
   pacman -S systemd
   ```

4. Enable systemd-networkd:

   ```bash
   systemctl enable systemd-networkd
   ```

5. Enable systemd-resolved:

   ```bash
   systemctl enable systemd-resolved
   ```

6. Create the network configuration directory:

   ```bash
   mkdir -p /etc/systemd/network/
   ```

7. Create a wireless network configuration file:

   ```bash
   nano /etc/systemd/network/25-wireless.network
   ```

   Paste the following:

   ```ini
   [Match]
   Name=wlan0

   [Network]
   DHCP=yes
   ```

   *(Note: Check your interface name with `ip link` if it's not `wlan0`.)*

8. Create the iwd configuration file:

   ```bash
   mkdir -p /etc/iwd/
   nano /etc/iwd/main.conf
   ```

   Paste the following:

   ```ini
   [General]
   EnableNetworkConfiguration=false

   [Network]
   NameResolvingService=systemd
   ```

9. Set up DNS resolution:

   ```bash
   ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
   ```

10. Reboot your system and use:

    ```bash
    iwctl
    ```

Your system now uses a minimal networking stack. systemd-networkd is used to support VPN and Ethernet.

---

**Tips for iwctl:**

* List devices:

  ```bash
  iwctl device list
  ```
* Scan networks:

  ```bash
  iwctl station <device> scan
  ```
* Get networks:

  ```bash
  iwctl station <device> get-networks
  ```
* Connect:

  ```bash
  iwctl station <device> connect <network-name>
  ```

---

## iwctlUsingRofi

Use Rofi to connect to Wi-Fi through iwctl.

1. Create your script:

   ```bash
   nano ~/scripts/Wi-Fi.sh
   ```

   Paste the following:

   ```bash
   #!/bin/bash

   # Detect Wi-Fi interface
   iface=$(iwctl device list | sed 's/\x1b\[[0-9;]*m//g' | awk '/wlan/ {print $1; exit}')

   # File to track manually saved SSIDs
   saved_file="$HOME/.config/iwctl-saved"
   mkdir -p ~/.config
   touch "$saved_file"

   # Get current SSID
   current=$(iwctl station "$iface" show | sed 's/\x1b\[[0-9;]*m//g' | grep 'Connected network' | awk '{print $NF}' | xargs)

   while true; do
     # Trigger new scan
     iwctl station "$iface" scan

     # Clean network list
     networks=$(iwctl station "$iface" get-networks |
       sed 's/\x1b\[[0-9;]*m//g' |
       grep -v "Available\|Network\|-----" |
       sed 's/>//g' |
       sed 's/^[ \t]*//' |
       awk '{print $1}' |
       grep -v "^$")

     # Build menu with refresh option at top
     menu=$(echo -e "[Refresh]\n$(echo "$networks" | while read -r line; do
       clean_line=$(echo "$line" | xargs)
       if [ "$clean_line" = "$current" ]; then
         echo "$clean_line (connected)"
       else
         echo "$clean_line"
       fi
     done)")

     # Rofi prompt
     chosen=$(echo "$menu" | rofi -dmenu -p "Select Wi-Fi")
     [ -z "$chosen" ] && exit 0

     chosen=$(echo "$chosen" | sed 's/ (connected)//')

     if [ "$chosen" = "[Refresh]" ]; then
       continue
     fi

     # Disconnect if selected again
     if [ "$chosen" = "$current" ]; then
       iwctl station "$iface" disconnect
       notify-send "Wi-Fi" "Disconnected from $chosen"
       exit 0
     fi

     # Check if SSID is in saved file
     if grep -Fxq "$chosen" "$saved_file"; then
       iwctl station "$iface" connect "$chosen"
       sleep 2
       new_current=$(iwctl station "$iface" show | sed 's/\x1b\[[0-9;]*m//g' | grep 'Connected network' | awk '{print $NF}' | xargs)
       if [ "$new_current" = "$chosen" ]; then
         notify-send "Wi-Fi" "Connected to $chosen"
         exit 0
       else
         sed -i "/^$chosen$/d" "$saved_file"
         notify-send "Wi-Fi" "Saved credentials for $chosen failed. Removed from list."
       fi
     fi

     # Prompt for password
     pass=$(echo "" | rofi -dmenu -password -p "Password for $chosen")
     if [ -n "$pass" ]; then
       iwctl station "$iface" connect "$chosen" --passphrase "$pass"
       sleep 2
       now=$(iwctl station "$iface" show | sed 's/\x1b\[[0-9;]*m//g' | grep 'Connected network' | awk '{print $NF}' | xargs)
       if [ "$now" = "$chosen" ]; then
         echo "$chosen" >> "$saved_file"
         sort -u "$saved_file" -o "$saved_file"
         notify-send "Wi-Fi" "Connected and saved $chosen"
       else
         notify-send "Wi-Fi" "Failed to connect to $chosen"
       fi
     else
       notify-send "Wi-Fi" "No password entered. Connection cancelled."
     fi

     break
   done
   ```

## OR

   ```bash
#!/bin/bash

# ============================================================================
# IWD ROFI WIFI MANAGER - KNOWN NETWORKS VERSION 1.5.0
# ============================================================================
# A complete, secure WiFi manager using iwd, busctl, and rofi
#
# Version: 1.5.0
# Date: 2025-01-24
#
# Changes from v1.4.1:
# - Replaced Favorites with automatic Known Networks tracking
# - Networks auto-saved on successful connection
# - Removed manual favorite starring (⭐)
# - [saved] tag now based on Known Networks file
# - Forget Network now removes from all storage locations
# - Known Networks file format: ssid|sec_type|device|timestamp
#
# SECURITY NOTE:
# Passwords passed via D-Bus are briefly visible in /proc/$PID/cmdline to
# root and same-user processes during the busctl call (microseconds). This
# is a limitation of command-line IPC and is significantly more secure than:
# - Shell history persistence
# - Filesystem credential storage
# - Cross-user visibility
#
# For maximum security, ensure proper PolicyKit rules are configured.
#
# Changes from v1.4.0:
# - Hardened signal_to_percent with regex validation
# - Fixed group check to properly split on newlines
# - Standardized printf '%b' instead of echo -e
# - Improved dbus_get_networks fallback parser
# - Added busctl -- separators for all signatures
# - Atomic favorites file writing
# - Extended connection timeout to 30s
# - Added --json mode for scripting
# - Removed eval from rfkill parsing
# - Consistent safe_label wrapping
# - Integer booleans in self_test
# ============================================================================

# ============================================================================
# ENVIRONMENT SETUP
# ============================================================================

export LC_ALL=C.UTF-8
export LANG=C.UTF-8

if ! locale -a 2>/dev/null | grep -qx 'C\.UTF-8'; then
export LC_ALL=C
export LANG=C
fi

set -eEuo pipefail

restore_terminal() {
 stty sane 2>/dev/null || true
 tput cnorm 2>/dev/null || true
}

trap 'restore_terminal; notify-send "Wi-Fi Error" "Unexpected error on line $LINENO"; exit 1' ERR INT TERM

# ============================================================================
# VERSION INFO
# ============================================================================

SCRIPT_VERSION="1.5.0"
SCRIPT_DATE="2025-01-24"

# ============================================================================
# DEBUG INFRASTRUCTURE
# ============================================================================

DEBUG_LOG="${XDG_STATE_HOME:-$HOME/.local/state}/iwd-rofi/debug.log"
mkdir -p "$(dirname "$DEBUG_LOG")"

debug_log() {
[ -n "${DEBUG:-}" ] || return 0

if [ ! -f "$DEBUG_LOG.pid" ] || [ "$(cat "$DEBUG_LOG.pid" 2>/dev/null)" != "$$" ]; then
 printf '\n========== NEW SESSION: %s (PID %s) ==========\n' "$(date '+%F %T')" "$$" >> "$DEBUG_LOG"
 echo "$$" > "$DEBUG_LOG.pid"
fi

local msg="$*"
msg=$(echo "$msg" | sed -E 's/(--passphrase[= ])[^ ]*/\1***REDACTED***/g')
msg=$(echo "$msg" | sed -E 's/(passphrase[= ])[^ ]*/\1***REDACTED***/g')
msg=$(echo "$msg" | sed -E 's/"s" "[^"]*"/"s" "***REDACTED***"/g')

printf '%s %s\n' "$(date '+%F %T')" "$msg" >> "$DEBUG_LOG"
}

BUS() {
local timeout=3
local safe_log=1

case "${1:-}" in
 call)
   case "${4:-}" in
     *Connect*|*Scan*|*GetOrderedNetworks*) timeout=15 ;;
   esac
   
   for arg in "$@"; do
     case "$arg" in
       Connect|RequestPassphrase|*passphrase*)
         safe_log=0
         break
         ;;
     esac
   done
   ;;
esac

if [ "${safe_log}" -eq 1 ]; then
 debug_log "busctl --no-pager --timeout=$timeout $*"
else
 debug_log "busctl [command with potential secrets - redacted]"
fi

command busctl --no-pager --timeout="$timeout" "$@"
}

IWCTL() {
debug_log "iwctl [command potentially redacted]"
iwctl "$@"
}

# ============================================================================
# CONFIGURATION & STORAGE
# ============================================================================

CFG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/iwd-rofi"
RUNTIME_DIR="${XDG_RUNTIME_DIR:-/tmp}"
LOCKFILE="$RUNTIME_DIR/iwd-rofi-$USER.lock"

mkdir -p "$CFG_DIR"
KNOWN_FILE="$CFG_DIR/known_networks"
HISTORY_FILE="$CFG_DIR/history"
touch "$KNOWN_FILE" "$HISTORY_FILE"
chmod 600 "$KNOWN_FILE" "$HISTORY_FILE" 2>/dev/null || true
chmod 700 "$CFG_DIR" 2>/dev/null || true

DBUS_SERVICE="net.connman.iwd"

DBUS_STATION_PATH=""
DBUS_DEVICE_PATH=""
IFACE=""

# ============================================================================
# ARGUMENT PARSING
# ============================================================================

ARGS=()
DEBUG=""
SELF_TEST=""
JSON_MODE=""

for arg in "$@"; do
case "$arg" in
 --debug)
   DEBUG=1
   ;;
 --self-test)
   SELF_TEST=1
   ;;
 --json)
   JSON_MODE=1
   ;;
 --version)
   echo "IWD Rofi Manager v$SCRIPT_VERSION ($SCRIPT_DATE)"
   exit 0
   ;;
 --help)
   cat <<EOF
IWD Rofi WiFi Manager v$SCRIPT_VERSION

USAGE:
$(basename "$0") [OPTIONS]

OPTIONS:
--toggle      Toggle WiFi on/off (CLI mode)
--scan        Trigger network scan (CLI mode)
--json        Output current state as JSON (CLI mode)
--self-test   Run diagnostics
--debug       Enable debug logging
--version     Show version
--help        Show this help

GUI MODE (default):
Interactive WiFi manager with rofi interface

Networks are automatically saved to Known Networks on successful connection.

EOF
   exit 0
   ;;
 *)
   ARGS+=("$arg")
   ;;
esac
done
set -- "${ARGS[@]}"

if [ -n "$DEBUG" ]; then
exec 9>>"$DEBUG_LOG"
export BASH_XTRACEFD=9
export PS4='[${LINENO}] $ '
set -x
debug_log "===== DEBUG ON (pid $$) ====="
fi

# ============================================================================
# DEPENDENCY VERIFICATION
# ============================================================================

missing_deps=()
for cmd in busctl rofi flock notify-send; do
 if ! command -v "$cmd" &>/dev/null; then
     missing_deps+=("$cmd")
 fi
done

if ! command -v iwctl &>/dev/null; then
 echo "Warning: iwctl not found - some features may be limited" >&2
fi

if [ ${#missing_deps[@]} -gt 0 ]; then
 error_msg="Missing required dependencies: ${missing_deps[*]}\n\nPlease install:\n"
 for dep in "${missing_deps[@]}"; do
     case "$dep" in
         busctl) error_msg+="- systemd (provides busctl)\n" ;;
         rofi) error_msg+="- rofi (menu interface)\n" ;;
         flock) error_msg+="- util-linux (provides flock)\n" ;;
         notify-send) error_msg+="- libnotify (provides notify-send)\n" ;;
     esac
 done
 printf '%b' "$error_msg" >&2
 command -v notify-send &>/dev/null && notify-send "Wi-Fi Manager Error" "$(printf '%b' "$error_msg")" -u critical || true
 exit 1
fi

# ============================================================================
# PERMISSION CHECK
# ============================================================================

if [ "$(id -u)" -ne 0 ]; then
 if ! id -nG | tr ' ' '\n' | grep -qxE 'network|wheel'; then
     error_msg="Insufficient permissions\n\nYou must be in the 'network' or 'wheel' group\n\nRun: sudo usermod -aG network $USER\nThen log out and back in"
     notify-send "Wi-Fi Error" "$(printf '%b' "$error_msg")" -u critical
     echo "Error: User must be in 'network' or 'wheel' group to control iwd" >&2
     exit 1
 fi
fi

# ============================================================================
# UI UTILITIES
# ============================================================================

escape_pango() {
 printf '%s' "$1" | sed 's/&/\&amp;/g; s/</\&lt;/g; s/>/\&gt;/g; s/"/\&quot;/g'
}

safe_label() {
 printf '%s' "$1" | tr -d '\000-\037\177'
}

pango_safe() {
 escape_pango "$(safe_label "$1")"
}

rofi_pick() {
 local prompt="$1"
 shift
 
 local status_msg=""
 if [ -n "$IFACE" ] && [ -n "$DBUS_STATION_PATH" ]; then
     local state
     state=$(dbus_get_station_state 2>/dev/null || echo "unknown")
     local connected_ssid
     connected_ssid=$(get_connected_ssid 2>/dev/null || true)
     
     if [ -n "$connected_ssid" ]; then
         local safe_connected
         safe_connected=$(pango_safe "$connected_ssid")
         status_msg="$IFACE — Connected to: $safe_connected"
     else
         status_msg="$IFACE — State: $state"
     fi
 fi
 
 rofi -dmenu -markup-rows -no-custom -i \
      -p "$prompt" \
      -no-fixed-num-lines \
      -eh 2 \
      -lines 15 \
      ${status_msg:+-mesg "$status_msg"} \
      "$@" || true
}

path_has_interface() {
local path="$1" iface="$2"
[ -n "$path" ] || return 1
BUS introspect "$DBUS_SERVICE" "$path" "$iface" >/dev/null 2>&1
}

# ============================================================================
# SIGNAL STRENGTH UTILITIES
# ============================================================================

signal_to_percent() {
 local signal_raw="${1:-}"
 local dbm percent
 
 [[ "$signal_raw" =~ ^-?[0-9]+$ ]] || { echo 0; return; }
 
 if (( signal_raw == 0 )); then
     echo 0
     return
 fi
 
 if (( signal_raw < -200 || signal_raw > 0 )); then
     dbm=$(( signal_raw / 100 ))
 else
     dbm=$signal_raw
 fi
 
 if (( dbm >= -30 )); then
     percent=100
 elif (( dbm <= -90 )); then
     percent=0
 else
     percent=$(( (dbm + 90) * 100 / 60 ))
 fi
 
 (( percent < 0 )) && percent=0
 (( percent > 100 )) && percent=100
 
 echo "$percent"
}

get_signal_icon() {
 local percent="$1"
 
 if [ "$percent" -ge 75 ]; then
     echo "▂▄▆█"
 elif [ "$percent" -ge 50 ]; then
     echo "▂▄▆_"
 elif [ "$percent" -ge 25 ]; then
     echo "▂▄__"
 else
     echo "▂___"
 fi
}

# ============================================================================
# SECURITY TYPE HANDLING
# ============================================================================

normalize_security() {
 local sec="$1"
 case "$sec" in
     psk) echo "WPA2" ;;
     sae) echo "WPA3" ;;
     open) echo "Open" ;;
     8021x) echo "Enterprise" ;;
     owe) echo "OWE" ;;
     *)
         printf '%s' "$sec" | tr '[:lower:]' '[:upper:]' | tr -d '\000-\037\177'
         ;;
 esac
}

sec_keyword() {
case "$1" in
 sae|SAE|wpa3|WPA3) echo "sae" ;;
 psk|PSK|wpa2|WPA2) echo "psk" ;;
 open|OPEN)         echo "open" ;;
 owe|OWE)           echo "owe" ;;
 8021x|EAP|eap)     echo "8021x" ;;
 *)                 echo "$1" ;;
esac
}

# ============================================================================
# IWCTL WRAPPER FUNCTIONS
# ============================================================================

has_iwctl() { command -v iwctl >/dev/null 2>&1; }

iwctl_scan() {
has_iwctl || return 1
IWCTL station "$IFACE" scan >/dev/null 2>&1
}

iwctl_disconnect() {
has_iwctl || return 1
IWCTL station "$IFACE" disconnect >/dev/null 2>&1
}

iwctl_power_set() {
has_iwctl && IWCTL device "$IFACE" set-property Powered "$1" >/dev/null 2>&1 && return 0
if command -v rfkill >/dev/null 2>&1; then
 if [ "$1" = "true" ]; then
   rfkill unblock wifi >/dev/null 2>&1 || true
 else
   rfkill block wifi   >/dev/null 2>&1 || true
 fi
 return 0
fi
return 1
}

get_link_rates() {
command -v iw >/dev/null 2>&1 || return 1
local tx="" rx=""
while IFS= read -r line; do
 case "$line" in
   *"tx bitrate"*)
     tx=$(printf '%s' "${line#*:}" | awk '{print $1" "$2}')
     ;;
   *"rx bitrate"*)
     rx=$(printf '%s' "${line#*:}" | awk '{print $1" "$2}')
     ;;
 esac
done < <(iw dev "$IFACE" link 2>/dev/null)

[ -z "$tx" ] && [ -z "$rx" ] && return 1

if [ -n "$tx" ] && [ -n "$rx" ]; then
 printf '%s' "TX $tx, RX $rx"
elif [ -n "$tx" ]; then
 printf '%s' "TX $tx"
else
 printf '%s' "RX $rx"
fi
}

# ============================================================================
# D-BUS: DEVICE OPERATIONS
# ============================================================================

dbus_get_device_powered() {
 BUS get-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Powered 2>/dev/null | awk '{print $2}'
}

dbus_set_device_powered() {
 local state="$1"
 BUS set-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Powered b "$state" >/dev/null 2>&1 && return 0
 iwctl_power_set "$state" && return 0
 return 1
}

# ============================================================================
# D-BUS: STATION STATE OPERATIONS
# ============================================================================

is_connected_state() {
 local state="$1"
 case "$state" in
     connected*|online*) return 0 ;;
     *) return 1 ;;
 esac
}

is_trying_to_connect() {
 case "$1" in
     associating*|configuring*|connecting*|roaming*) return 0 ;;
     connected*|online*|disconnected*|idle*|scanning*) return 1 ;;
     *) return 1 ;;
 esac
}

dbus_get_station_state() {
 BUS get-property "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station State 2>/dev/null | cut -d'"' -f2
}

dbus_get_connected_network_path() {
 local path
 path=$(BUS get-property "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station ConnectedNetwork 2>/dev/null | awk '{print $2}' | tr -d '"')
 
 if [ -z "$path" ] || [ "$path" = "/" ]; then
     echo ""
 else
     echo "$path"
 fi
}

dbus_scan_networks() {
 BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station Scan >/dev/null 2>&1 && return 0
 iwctl_scan && return 0
 return 1
}

dbus_disconnect() {
 BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station Disconnect >/dev/null 2>&1 && return 0
 iwctl_disconnect && return 0
 return 1
}

get_connected_ssid() {
local p
p=$(dbus_get_connected_network_path) || true
[ -z "$p" ] && { echo ""; return 0; }
BUS get-property "$DBUS_SERVICE" "$p" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2
}

# ============================================================================
# D-BUS: NETWORK ENUMERATION
# ============================================================================

list_network_paths_under_station() {
local station="$1"
BUS tree "$DBUS_SERVICE" --list 2>/dev/null \
 | awk -v P="$station" 'index($0,P"/")==1 && /_psk$|_8021x$|\/network/ {print}'
}

dbus_get_networks() {
 local service="$DBUS_SERVICE"
 local -A networks=()
 local -A seen_keys=()

 if command -v jq >/dev/null 2>&1; then
     local json_result
     if json_result=$(BUS call --json=short "$service" "$DBUS_STATION_PATH" net.connman.iwd.Station GetOrderedNetworks 2>/dev/null); then
         while IFS=$'\t' read -r path signal; do
             [ -z "$path" ] && continue
             local name sec
             name=$(BUS get-property "$service" "$path" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2)
             sec=$(BUS get-property "$service" "$path" net.connman.iwd.Network Type 2>/dev/null | cut -d'"' -f2)
             [ -z "$name" ] && continue
             [ -z "$sec" ] && sec="open"
             local key="${name}|${sec}"
             if [ -z "${seen_keys[$key]:-}" ]; then
                 seen_keys["$key"]=1
                 local dbm=$(( signal / 100 ))
                 local percent; percent=$(signal_to_percent "$dbm")
                 printf '%s|%s|%s|%s\n' "$path" "$name" "$percent" "$sec"
             fi
         done < <(echo "$json_result" | jq -r '.data[][] | select(.[0] != null) | @tsv' 2>/dev/null) | sort -t'|' -k3 -rn
         return 0
     fi
 fi

 local ordered_output
 ordered_output=$(BUS call "$service" "$DBUS_STATION_PATH" net.connman.iwd.Station GetOrderedNetworks 2>/dev/null || true)

 if [ -z "$ordered_output" ]; then
     while IFS= read -r p; do
         [ -z "$p" ] && continue
         local name sec
         name=$(BUS get-property "$service" "$p" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2)
         sec=$(BUS get-property "$service" "$p" net.connman.iwd.Network Type 2>/dev/null | cut -d'"' -f2)
         [ -z "$name" ] && continue
         [ -z "$sec" ] && sec="open"
         local sig dbm percent
         sig=$(BUS get-property "$service" "$p" net.connman.iwd.Network Signal 2>/dev/null | awk '{print $2}')
         if [[ "$sig" =~ ^-?[0-9]+$ ]]; then
             dbm=$(( sig / 100 ))
             percent=$(signal_to_percent "$dbm")
         else
             percent=0
         fi
         printf '%s|%s|%s|%s\n' "$p" "$name" "$percent" "$sec"
     done < <(list_network_paths_under_station "$DBUS_STATION_PATH") \
     | sort -t'|' -k3 -rn
     return 0
 fi

 local expect="path" cur_path=""
 while IFS= read -r tok; do
     [ -z "$tok" ] && continue
     if [ "$expect" = "path" ]; then
         if [[ $tok =~ \"(/net/connman/iwd/[^\"]+)\" ]]; then
             cur_path="${BASH_REMATCH[1]}"; expect="signal"
         fi
     else
         if [[ "$tok" =~ (-?[0-9]+) ]]; then
             local cent="${BASH_REMATCH[1]}"
             if [ -n "$cur_path" ]; then
                 local name sec
                 name=$(BUS get-property "$service" "$cur_path" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2)
                 sec=$(BUS get-property "$service" "$cur_path" net.connman.iwd.Network Type 2>/dev/null | cut -d'"' -f2)
                 [ -z "$name" ] && { expect="path"; cur_path=""; continue; }
                 [ -z "$sec" ] && sec="open"
                 local key="${name}|${sec}"
                 if [ -z "${seen_keys[$key]:-}" ]; then
                     seen_keys["$key"]=1
                     networks["$cur_path"]="$cent|$name|$sec"
                 fi
             fi
         fi
         expect="path"; cur_path=""
     fi
 done < <(printf '%s\n' "$ordered_output" | sed 's/[ \t]\+/\n/g')

 [ "${#networks[@]}" -eq 0 ] && return 1
 for p in "${!networks[@]}"; do
     local cent name sec
     IFS='|' read -r cent name sec <<< "${networks[$p]}"
     local dbm=$(( cent / 100 ))
     local percent; percent=$(signal_to_percent "$dbm")
     printf '%s|%s|%s|%s\n' "$p" "$name" "$percent" "$sec"
 done | sort -t'|' -k3 -rn
}

# ============================================================================
# D-BUS: NETWORK CONNECTION OPERATIONS
# ============================================================================

ensure_station() {
 has_iwctl || return 1
 IWCTL station "$IFACE" show >/dev/null 2>&1
}

get_best_bssid_for_group() {
 local group_path="$1" 
 local service="$DBUS_SERVICE"
 local best_bssid="" best_signal=-32768
 
 if BUS call "$service" "$group_path" net.connman.iwd.Network GetOrderedBSSes >/dev/null 2>&1; then
     local bss_list
     bss_list=$(BUS call "$service" "$group_path" net.connman.iwd.Network GetOrderedBSSes 2>/dev/null | grep -o '"/[^"]*"' | head -1 | tr -d '"')
     if [ -n "$bss_list" ]; then
         best_bssid=$(BUS get-property "$service" "$bss_list" net.connman.iwd.BSS Address 2>/dev/null | cut -d'"' -f2)
         [ -n "$best_bssid" ] && { echo "$best_bssid"; return 0; }
     fi
 fi
 
 while IFS= read -r bss; do
     [ -z "$bss" ] && continue
     local mac signal
     mac=$(BUS get-property "$service" "$bss" net.connman.iwd.BSS Address 2>/dev/null | cut -d'"' -f2) || continue
     signal=$(BUS get-property "$service" "$bss" net.connman.iwd.BSS Signal 2>/dev/null | awk '{print $2}') || continue
     
     if [ -n "$mac" ] && [ "$signal" -gt "$best_signal" ]; then
         best_signal="$signal"
         best_bssid="$mac"
     fi
 done < <(BUS tree "$service" --list 2>/dev/null | awk -v p="$group_path" 'index($0, p"/")==1')
 
 [ -n "$best_bssid" ] && echo "$best_bssid"
}

known_network_exists_for_ssid() {
local ssid="$1"
BUS tree "$DBUS_SERVICE" --list 2>/dev/null \
 | while read -r p; do
     [ -z "$p" ] && continue
     BUS introspect "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork >/dev/null 2>&1 || continue
     local nm
     nm=$(BUS get-property "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork Name 2>/dev/null | cut -d'"' -f2)
     [ "$nm" = "$ssid" ] && { echo 1; break; }
   done
}

dbus_connect_network() {
 local group_path="$1"
 local passphrase="$2"

 local ssid connect_sig
 ssid=$(BUS get-property "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2) || ssid=""
 connect_sig="$(BUS introspect "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network 2>/dev/null | awk '$1==".Connect"{print $3; exit}')"

 if { [ -z "$connect_sig" ] || [ "$connect_sig" = "-" ]; } && [ -n "$passphrase" ]; then
     ensure_station || return 1
     IWCTL --passphrase="$passphrase" station "$IFACE" connect "$ssid" >/dev/null 2>&1 && return 0
 fi

 if [ -z "$passphrase" ]; then
     if [ -n "$(known_network_exists_for_ssid "$ssid")" ] || [ "$connect_sig" = "-" ]; then
         BUS call "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Connect >/dev/null 2>&1 && return 0
     fi
 else
     case "$connect_sig" in
         s)
             BUS call "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Connect s -- "$passphrase" >/dev/null 2>&1 && return 0
             ;;
         a*)
             BUS call "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Connect a{sv} 1 "Passphrase" s "$passphrase" >/dev/null 2>&1 && return 0
             ;;
         -|"")
             : 
             ;;
     esac
 fi

 ensure_station || return 1
 if [ -z "$passphrase" ]; then
     IWCTL station "$IFACE" connect "$ssid" >/dev/null 2>&1 && return 0
 else
     IWCTL --passphrase="$passphrase" station "$IFACE" connect "$ssid" >/dev/null 2>&1 && return 0
 fi

 return 1
}

station_supports() {
local method="$1"
BUS introspect "$DBUS_SERVICE" "$DBUS_STATION_PATH" 2>/dev/null \
 | grep -q " $method("
}

find_known_network_path_by_name() {
local want="$1" 
local p name
while IFS= read -r p; do
 [ -z "$p" ] && continue
 BUS introspect "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork >/dev/null 2>&1 || continue
 name=$(BUS get-property "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork Name 2>/dev/null | cut -d'"' -f2)
 [ "$name" = "$want" ] && { printf '%s\n' "$p"; return 0; }
done < <(BUS tree "$DBUS_SERVICE" --list 2>/dev/null | grep '^/net/connman/iwd/')
return 1
}

list_knownnetwork_paths_for_ssid() {
local ssid="$1"
BUS tree "$DBUS_SERVICE" --list 2>/dev/null \
 | while read -r p; do
     [ -z "$p" ] && continue
     BUS introspect "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork >/dev/null 2>&1 || continue
     local nm
     nm=$(BUS get-property "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork Name 2>/dev/null | cut -d'"' -f2 || true)
     [ -n "$nm" ] && [ "$nm" = "$ssid" ] && printf '%s\n' "$p"
   done
}

dbus_connect_hidden_network() {
local ssid="$1" pass="$2" sec_raw="$3"
local sec
sec="$(sec_keyword "$sec_raw")"

if station_supports "ConnectHiddenNetwork"; then
 local n=0 
 local -a args=()
 [ -n "$sec" ]  && { args+=("Type" s "$sec"); n=$((n+1)); }
 if [ -n "$pass" ] && [ "$sec" != "open" ] && [ "$sec" != "owe" ]; then
   args+=("Passphrase" s "$pass"); n=$((n+1))
 fi
 BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station ConnectHiddenNetwork \
   -- s "$ssid" a{sv} $n "${args[@]}" >/dev/null 2>&1 && return 0
fi

local kn_path
kn_path="$(find_known_network_path_by_name "$ssid" || true)"
if [ -n "$kn_path" ] && station_supports "ConnectKnownNetwork"; then
 BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station ConnectKnownNetwork -- o "$kn_path" \
   >/dev/null 2>&1 && return 0
fi

return 1
}

forget_network_by_ssid() {
local ssid="$1"
local known_path="${2:-}"

local deleted=0

if [ -n "$known_path" ]; then
 BUS call "$DBUS_SERVICE" "$known_path" net.connman.iwd.KnownNetwork Forget >/dev/null 2>&1 && deleted=1 || true
fi

while IFS= read -r p; do
 [ -z "$p" ] && continue
 BUS call "$DBUS_SERVICE" "$p" net.connman.iwd.KnownNetwork Forget >/dev/null 2>&1 && deleted=1 || true
done < <(list_knownnetwork_paths_for_ssid "$ssid")

{
 local raw="$ssid"
 local san
 san=$(printf '%s' "$ssid" | iconv -t ASCII//TRANSLIT 2>/dev/null | sed 's/[^-_.[:alnum:]]/_/g' || printf '%s' "$ssid")
 rm -f "/var/lib/iwd/${raw}.psk" "/var/lib/iwd/${raw}.8021x" \
       "/var/lib/iwd/${san}.psk" "/var/lib/iwd/${san}.8021x"
} >/dev/null 2>&1 || true

if has_iwctl; then
 IWCTL known-networks "$ssid" forget >/dev/null 2>&1 && deleted=1 || true
fi

return 0
}

# ============================================================================
# KNOWN NETWORKS MANAGEMENT
# ============================================================================

is_known_network() {
    local ssid="$1" sec_type="$2" device="$3"
    awk -F'|' -v s="$ssid" -v t="$sec_type" -v d="$device" '
        $1 == s && $2 == t && $3 == d {found=1; exit}
        END {exit !found}
    ' "$KNOWN_FILE" 2>/dev/null
}

record_known_network() {
    local ssid="$1" sec_type="$2" device="$3"
    local timestamp="${4:-$(date '+%Y-%m-%d %H:%M:%S')}"

    (
        flock -x 200

        awk -F'|' -v s="$ssid" -v t="$sec_type" -v d="$device" -v ts="$timestamp" '
            BEGIN { updated = 0 }
            $1 == s && $2 == t && $3 == d {
                print s "|" t "|" d "|" ts
                updated = 1
                next
            }
            { print $0 }
            END {
                if (!updated) {
                    print s "|" t "|" d "|" ts
                }
            }
        ' "$KNOWN_FILE" 2>/dev/null > "$KNOWN_FILE.tmp"

        mv "$KNOWN_FILE.tmp" "$KNOWN_FILE"
    ) 200>"$KNOWN_FILE.lock"
}

remove_known_network() {
    local ssid="$1" sec_type="$2" device="$3"
    
    (
        flock -x 200
        awk -F'|' -v s="$ssid" -v t="$sec_type" -v d="$device" '
            !($1 == s && $2 == t && $3 == d)
        ' "$KNOWN_FILE" > "$KNOWN_FILE.tmp" 2>/dev/null || :
        mv "$KNOWN_FILE.tmp" "$KNOWN_FILE"
    ) 200>"$KNOWN_FILE.lock"
}

# ============================================================================
# KNOWN NETWORK HEALTH + REPAIR CONNECT
# ============================================================================

is_knownnetwork_healthy() {
local ssid="$1"
local kn
kn="$(find_known_network_path_by_name "$ssid" 2>/dev/null || true)"
[ -z "$kn" ] && return 0
BUS get-property "$DBUS_SERVICE" "$kn" net.connman.iwd.KnownNetwork Name >/dev/null 2>&1
}

attempt_connect_with_repair() {
local net_path="$1" ssid="$2" pass="$3"

local kn
kn="$(find_known_network_path_by_name "$ssid" 2>/dev/null || true)"
if [ -n "$pass" ] && [ -n "$kn" ]; then
 forget_network_by_ssid "$ssid" "" || true
 sleep 1
fi

if dbus_connect_network "$net_path" "$pass"; then
 wait_for_connection "$net_path" "$ssid"; rc=$?
 [ $rc -eq 0 ] && return 0

 if [ $rc -eq 2 ]; then
   forget_network_by_ssid "$ssid" "" || true
   sleep 1
   if dbus_connect_network "$net_path" "$pass"; then
     wait_for_connection "$net_path" "$ssid" && return 0
   fi
 fi
fi

return 1
}

wait_for_connection() {
 local expected_path="$1"
 local expected_ssid="$2"
 local max_wait=30
 local wait_iter=0
 local fail_count=0
 
 while (( wait_iter < max_wait )); do
     wait_iter=$((wait_iter + 1))
     
     local state p name
     state=$(dbus_get_station_state 2>/dev/null || echo "unknown")
     p=$(dbus_get_connected_network_path 2>/dev/null || true)
     name=$(get_connected_ssid 2>/dev/null || true)
     
     if { [ -n "$expected_path" ] && [ "$p" = "$expected_path" ]; } \
        || { [ -z "$expected_path" ] && [ -n "$name" ] && [ "$name" = "$expected_ssid" ]; }; then
         if is_connected_state "$state"; then
             local timestamp
             timestamp="$(date '+%Y-%m-%d %H:%M:%S')"
             
             (
                 flock -x 201
                 echo "${p:-/}|$expected_ssid|$timestamp" >> "$HISTORY_FILE"
             ) 201>"$HISTORY_FILE.lock"
             
             local sec_type
             if [ -n "$p" ]; then
                 sec_type=$(BUS get-property "$DBUS_SERVICE" "$p" net.connman.iwd.Network Type 2>/dev/null | cut -d'"' -f2 || echo "psk")
             else
                 sec_type="psk"
             fi
             record_known_network "$expected_ssid" "$sec_type" "$IFACE" "$timestamp"
             
             notify-send "Wi-Fi" "Connected to $(safe_label "$expected_ssid")"
             return 0
         fi
     fi
     
     case "$state" in
         disconnected*|idle*)
             fail_count=$((fail_count + 1))
             if (( fail_count >= 3 && wait_iter > 5 )); then
                 notify-send "Wi-Fi" "Authentication failed for $(safe_label "$expected_ssid")"
                 return 2
             fi
             ;;
     esac
     
     sleep 1
 done
 
 notify-send "Wi-Fi" "Failed to connect to $(safe_label "$expected_ssid")"
 return 1
}

# ============================================================================
# NETWORK DETAILS VIEW
# ============================================================================

show_network_details() {
 local net_path="${1:-}"
 local ssid_in="${2:-}"

 local mode=""
 if path_has_interface "$net_path" net.connman.iwd.Network; then
     mode="network"
 elif path_has_interface "$net_path" net.connman.iwd.KnownNetwork; then
     mode="known"
 else
     if [ -n "$ssid_in" ]; then
         local kn_path
         kn_path="$(find_known_network_path_by_name "$ssid_in" 2>/dev/null || true)"
         if [ -n "$kn_path" ] && path_has_interface "$kn_path" net.connman.iwd.KnownNetwork; then
             net_path="$kn_path"
             mode="known"
         else
             local live_path
             live_path="$(find_network_by_ssid "$ssid_in" "$(dbus_get_networks 2>/dev/null || true)" || true)"
             if [ -n "$live_path" ] && path_has_interface "$live_path" net.connman.iwd.Network; then
                 net_path="$live_path"
                 mode="network"
             fi
         fi
     fi
 fi

 if [ -z "$mode" ]; then
     local safe_ssid
     safe_ssid="$(pango_safe "${ssid_in:-Unknown}")"
     local details="<b>Network:</b> $safe_ssid
No further details available (AP out of range and not saved)."
     printf '%b' "Close\n" | rofi -dmenu -markup-rows -p "Network Details" -mesg "$details"
     return 0
 fi

 local details=""
 local safe_ssid=""
 local ip=""
 local last_connect=""
 local current_path=""

 if [ "$mode" = "known" ]; then
     local kn_name
     kn_name="$(BUS get-property "$DBUS_SERVICE" "$net_path" net.connman.iwd.KnownNetwork Name 2>/dev/null | cut -d'"' -f2 || true)"
     safe_ssid="$(pango_safe "${kn_name:-$ssid_in}")"
     details="<b>Network (saved):</b> $safe_ssid
<i>Out of range or not currently visible.</i>"
 else
     local name sec_type signal
     name="$(BUS get-property "$DBUS_SERVICE" "$net_path" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2 || true)"
     sec_type="$(BUS get-property "$DBUS_SERVICE" "$net_path" net.connman.iwd.Network Type 2>/dev/null | cut -d'"' -f2 || true)"
     signal="$(BUS get-property "$DBUS_SERVICE" "$net_path" net.connman.iwd.Network Signal 2>/dev/null | awk '{print $2}' || true)"

     safe_ssid="$(pango_safe "${name:-$ssid_in}")"
     details="<b>Network:</b> $safe_ssid"
     if [ -n "$sec_type" ]; then
         details="$details
<b>Security:</b> $(normalize_security "$sec_type")"
     fi

     if [[ "$signal" =~ ^-?[0-9]+$ ]]; then
         local dbm=$(( signal / 100 ))
         local percent
         percent="$(signal_to_percent "$dbm")"
         details="$details
<b>Signal:</b> ${dbm} dBm (${percent}%)"
     fi

     current_path="$(dbus_get_connected_network_path 2>/dev/null || true)"
     if [ -n "$current_path" ] && [ "$net_path" = "$current_path" ]; then
         details="$details
<b>Status:</b> Connected"
         ip="$(ip -4 addr show "$IFACE" 2>/dev/null | awk '/inet /{print $2}' | cut -d/ -f1 | head -1 || true)"
         if [ -n "$ip" ]; then
             details="$details
<b>IP Address:</b> $ip"
         fi
         last_connect="$(get_history "$net_path" | head -1 | cut -d'|' -f3 || true)"
         if [ -n "$last_connect" ]; then
             details="$details
<b>Last Connected:</b> $last_connect"
         fi
     local lr
         lr="$(get_link_rates 2>/dev/null || true)"
         if [ -n "$lr" ]; then
             details="$details
<b>Link Rate:</b> $lr"
         fi
     fi
 fi

printf '%b' "Close\n" | rofi -dmenu -markup-rows -p "Network Details" -mesg "$details"
}

# ============================================================================
# RFKILL STATUS CHECK
# ============================================================================

check_rfkill_status() {
 command -v rfkill >/dev/null 2>&1 || return 0
 
 local out soft_blocked hard_blocked
 out="$(rfkill list wifi 2>/dev/null || rfkill list 2>/dev/null || true)"
 soft_blocked=$(printf '%s\n' "$out" | grep -Eo 'Soft blocked: (yes|no)' | awk '{print $3}')
 hard_blocked=$(printf '%s\n' "$out" | grep -Eo 'Hard blocked: (yes|no)' | awk '{print $3}')
 
 if [ "$hard_blocked" = "yes" ]; then
     notify-send "Wi-Fi" "Hardware RF-Kill is ON\n\nCheck physical Wi-Fi switch/button" -u critical
     return 1
 fi
 
 if [ "$soft_blocked" = "yes" ]; then
     notify-send "Wi-Fi" "Software RF-Kill is ON\n\nTry: rfkill unblock wifi" -u normal
     return 1
 fi
 
 return 0
}

# ============================================================================
# CONNECTION HISTORY
# ============================================================================

get_history() {
 local net_path="$1"
 awk -F'|' -v p="$net_path" '
     $1 == p && NF >= 3 && $2 != "" {print}
 ' "$HISTORY_FILE" 2>/dev/null | sort -t'|' -k3 -r || true
}

clear_history() {
local net_path="$1"
awk -F'|' -v p="$net_path" '$1!=p' "$HISTORY_FILE" 2>/dev/null > "$HISTORY_FILE.tmp" || :
mv "$HISTORY_FILE.tmp" "$HISTORY_FILE"
}

# ============================================================================
# NETWORK SEARCH
# ============================================================================

find_network_by_ssid() {
 local target_ssid="$1"
 local network_list="$2"
 
 while IFS='|' read -r net_path ssid signal_percent sec_type; do
     [ -z "$net_path" ] && continue
     if [ "$ssid" = "$target_ssid" ]; then
         echo "$net_path"
         return 0
     fi
 done <<< "$network_list"
 
 return 1
}

# ============================================================================
# DYNAMIC PATH DISCOVERY
# ============================================================================

discover_iwd_paths() {
 local service="$DBUS_SERVICE"
 local all_paths
 local -a candidate_stations_array=()
 local best_station parent_path
 
 all_paths=$(BUS tree "$service" --list 2>/dev/null | grep '^/net/connman/iwd/' || true)
 [ -z "$all_paths" ] && return 1
 
 while IFS= read -r obj; do
     [ -z "$obj" ] && continue
     if BUS introspect "$service" "$obj" net.connman.iwd.Station >/dev/null 2>&1; then
         candidate_stations_array+=("$obj")
     fi
 done <<< "$all_paths"
 
 best_station=""
 for candidate in "${candidate_stations_array[@]}"; do
     if BUS call "$service" "$candidate" net.connman.iwd.Station GetOrderedNetworks >/dev/null 2>&1; then
         best_station="$candidate"
         break
     fi
 done
 
 [ -z "$best_station" ] && return 1
 DBUS_STATION_PATH="$best_station"
 
 if BUS introspect "$service" "$DBUS_STATION_PATH" net.connman.iwd.Device >/dev/null 2>&1; then
     DBUS_DEVICE_PATH="$DBUS_STATION_PATH"
 else
     parent_path="${DBUS_STATION_PATH%/*}"
     if BUS introspect "$service" "$parent_path" net.connman.iwd.Device >/dev/null 2>&1; then
         DBUS_DEVICE_PATH="$parent_path"
     else
         local sib
         while IFS= read -r sib; do
             [ -z "$sib" ] && continue
             if BUS introspect "$service" "$sib" net.connman.iwd.Device >/dev/null 2>&1; then
                 DBUS_DEVICE_PATH="$sib"
                 break
             fi
         done < <(BUS tree "$service" --list 2>/dev/null | awk -v p="$parent_path" '
             index($0, p"/")==1 {
                 rest = substr($0, length(p)+2)
                 if (index(rest, "/")==0) print
             }'
         )
     fi
 fi
 
 [ -n "$DBUS_DEVICE_PATH" ] && return 0
 return 1
}

get_iface_name() {
 local name
 name=$(BUS get-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Name 2>/dev/null | cut -d'"' -f2)
 
 [ -n "$name" ] && { echo "$name"; return 0; }
 echo "wlan0"
}

# ============================================================================
# JSON OUTPUT UTILITIES
# ============================================================================

json_escape() {
    local s="$1"
    s=${s//\\/\\\\}
    s=${s//\"/\\\"}
    s=${s//$'\n'/\\n}
    s=${s//$'\r'/\\r}
    s=${s//$'\t'/\\t}
    printf '%s' "$s"
}

print_json_state() {
 local state connected_path connected_ssid
 state=$(dbus_get_station_state 2>/dev/null || echo "")
 connected_path=$(dbus_get_connected_network_path 2>/dev/null || echo "")
 connected_ssid=$(get_connected_ssid 2>/dev/null || echo "")
 
 local list
 list=$(dbus_get_networks 2>/dev/null || echo "")
 
 printf '{'
 printf '"iface":"%s",' "$(json_escape "$IFACE")"
 printf '"state":"%s",' "$(json_escape "$state")"
 printf '"powered":%s,' "$( [ "$(dbus_get_device_powered 2>/dev/null)" = "true" ] && echo true || echo false )"
 printf '"connected_ssid":"%s",' "$(json_escape "$connected_ssid")"
 printf '"networks":['
 
 local first=1
 while IFS='|' read -r p ssid percent sec; do
     [ -z "$p" ] && continue
     if [ $first -eq 0 ]; then
         printf ','
     fi
     first=0
     printf '{'
     printf '"path":"%s",' "$(json_escape "$p")"
     printf '"ssid":"%s",' "$(json_escape "$ssid")"
     printf '"percent":%s,' "$percent"
     printf '"sec":"%s"' "$(json_escape "$sec")"
     printf '}'
 done <<< "$list"
 
 printf ']}'
}

# ============================================================================
# ENHANCED SELF-TEST DIAGNOSTICS
# ============================================================================

self_test() {
 set +e
 
 local report=""
 local all_good=0
 
 report+="<b>=== Wi-Fi Manager Diagnostics v$SCRIPT_VERSION ===</b>\n\n"
 
 report+="<b>File Permissions:</b>\n"
 if [ -w "$HISTORY_FILE" ]; then
     report+="  ✓ History file writable\n"
 else
     report+="  ✗ History file not writable: $HISTORY_FILE\n"
     all_good=1
 fi
 
 if [ -w "$KNOWN_FILE" ]; then
     report+="  ✓ Known networks file writable\n"
 else
     report+="  ✗ Known networks file not writable: $KNOWN_FILE\n"
     all_good=1
 fi
 
 if [ -w "$(dirname "$DEBUG_LOG")" ]; then
     report+="  ✓ Debug log directory writable\n"
 else
     report+="  ✗ Debug log directory not writable\n"
     all_good=1
 fi
 
 report+="\n<b>System Services:</b>\n"
 local iwd_status
 iwd_status=$(systemctl is-active iwd 2>/dev/null || echo "unknown")
 if [ "$iwd_status" = "active" ]; then
     report+="  ✓ IWD service: $iwd_status\n"
 else
     report+="  ✗ IWD service: $iwd_status\n"
     all_good=1
 fi
 
 local iwd_version
 iwd_version=$(iwctl --version 2>/dev/null | head -1 || echo "unknown")
 report+="  • IWD version: $iwd_version\n"
 
 report+="\n<b>User Permissions:</b>\n"
 local user_groups
 user_groups=$(id -nG)
 if echo "$user_groups" | tr ' ' '\n' | grep -qxE 'network|wheel'; then
     report+="  ✓ User in required group\n"
 else
     report+="  ✗ User not in network/wheel group\n"
     report+="    Run: sudo usermod -aG network $USER\n"
     all_good=1
 fi
 report+="  • Groups: $user_groups\n"
 
 report+="\n<b>Device Discovery:</b>\n"
 if discover_iwd_paths; then
     report+="  ✓ Found Station: $DBUS_STATION_PATH\n"
     report+="  ✓ Found Device: $DBUS_DEVICE_PATH\n"
     
     local iface_name
     iface_name=$(get_iface_name)
     report+="  • Interface: $iface_name\n"
     
     local powered
     powered=$(dbus_get_device_powered 2>/dev/null || echo "unknown")
     if [ "$powered" = "true" ]; then
         report+="  ✓ Device powered: ON\n"
     else
         report+="  ✗ Device powered: OFF\n"
     fi
     
     local state
     state=$(dbus_get_station_state 2>/dev/null || echo "unknown")
     report+="  • Station state: $state\n"
     
     local connected
     connected=$(get_connected_ssid 2>/dev/null || echo "none")
     report+="  • Connected to: $connected\n"
 else
     report+="  ✗ Failed to discover IWD paths\n"
     all_good=1
 fi
 
 report+="\n<b>RF-Kill Status:</b>\n"
 if command -v rfkill >/dev/null 2>&1; then
     local rf_output
     rf_output=$(rfkill list wifi 2>/dev/null)
     if echo "$rf_output" | grep -q "Soft blocked: yes"; then
         report+="  ✗ Soft blocked: YES\n"
         report+="    Run: rfkill unblock wifi\n"
         all_good=1
     else
         report+="  ✓ Soft blocked: NO\n"
     fi
     
     if echo "$rf_output" | grep -q "Hard blocked: yes"; then
         report+="  ✗ Hard blocked: YES (check physical switch)\n"
         all_good=1
     else
         report+="  ✓ Hard blocked: NO\n"
     fi
 else
     report+="  • rfkill not available\n"
 fi
 
 if [ -n "$DBUS_STATION_PATH" ]; then
     report+="\n<b>D-Bus API Tests:</b>\n"
     
     if BUS get-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Powered >/dev/null 2>&1; then
         report+="  ✓ Device.Powered property accessible\n"
     else
         report+="  ✗ Device.Powered property failed\n"
         all_good=1
     fi
     
     if BUS get-property "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station State >/dev/null 2>&1; then
         report+="  ✓ Station.State property accessible\n"
     else
         report+="  ✗ Station.State property failed\n"
         all_good=1
     fi
     
     if BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station GetOrderedNetworks >/dev/null 2>&1; then
         report+="  ✓ GetOrderedNetworks call successful\n"
         local net_count
         net_count=$(dbus_get_networks 2>/dev/null | wc -l)
         report+="  • Visible networks: $net_count\n"
     else
         report+="  ✗ GetOrderedNetworks call failed\n"
         all_good=1
     fi
 fi
 
 report+="\n<b>Required Dependencies:</b>\n"
 for cmd in busctl rofi flock notify-send; do
     if command -v "$cmd" >/dev/null 2>&1; then
         report+="  ✓ $cmd\n"
     else
         report+="  ✗ $cmd (missing)\n"
         all_good=1
     fi
 done
 
 report+="\n<b>Optional Dependencies:</b>\n"
 for cmd in iwctl rfkill jq; do
     if command -v "$cmd" >/dev/null 2>&1; then
         report+="  ✓ $cmd\n"
     else
         report+="  • $cmd (optional, not found)\n"
     fi
 done
 
 report+="\n<b>Summary:</b>\n"
 if (( all_good == 0 )); then
     report+="  ✓ All checks passed\n"
     printf '%b' "Close\n" | rofi -dmenu -markup-rows -p "Diagnostics: PASS" -mesg "$report"
     notify-send "Wi-Fi Self-Test" "All checks passed ✓" -u normal
     return 0
 else
     report+="  ✗ Some issues detected (see above)\n"
     printf '%b' "Close\n" | rofi -dmenu -markup-rows -p "Diagnostics: ISSUES FOUND" -mesg "$report"
     notify-send "Wi-Fi Self-Test" "Issues detected - see details" -u critical
     return 1
 fi
}

# ============================================================================
# CLI MODE: TOGGLE
# ============================================================================

if [ "${1:-}" = "--toggle" ]; then
 exec 200>"$LOCKFILE"
 flock -n 200 || { echo "Another instance is running"; exit 1; }
 
 all_paths=$(BUS tree "$DBUS_SERVICE" --list 2>/dev/null | grep '^/net/connman/iwd/' || true)
 [ -z "$all_paths" ] && { echo "No iwd paths found"; exit 1; }
 
 device_path=""
 while IFS= read -r obj; do
     [ -z "$obj" ] && continue
     if BUS introspect "$DBUS_SERVICE" "$obj" net.connman.iwd.Device >/dev/null 2>&1; then
         device_path="$obj"
         break
     fi
 done <<< "$all_paths"
 
 [ -z "$device_path" ] && { echo "No Device interface found"; exit 1; }
 
 DBUS_DEVICE_PATH="$device_path"
 
 current=$(dbus_get_device_powered)
 if [ "$current" = "true" ]; then
     dbus_set_device_powered false || { echo "Disable failed (DBus/iwctl/rfkill)"; exit 1; }
     echo "WiFi disabled"
 else
     dbus_set_device_powered true || { echo "Enable failed (DBus/iwctl/rfkill)"; exit 1; }
     echo "WiFi enabled"
 fi
 exit 0
fi

# ============================================================================
# CLI MODE: SCAN
# ============================================================================

if [ "${1:-}" = "--scan" ]; then
 exec 200>"$LOCKFILE"
 flock -n 200 || { echo "Another instance is running"; exit 1; }
 
 all_paths=$(BUS tree "$DBUS_SERVICE" --list 2>/dev/null | grep '^/net/connman/iwd/' || true)
 [ -z "$all_paths" ] && { echo "No iwd paths found"; exit 1; }
 
 station_path=""
 while IFS= read -r obj; do
     [ -z "$obj" ] && continue
     if BUS introspect "$DBUS_SERVICE" "$obj" net.connman.iwd.Station >/dev/null 2>&1; then
         if BUS call "$DBUS_SERVICE" "$obj" net.connman.iwd.Station GetOrderedNetworks >/dev/null 2>&1; then
             station_path="$obj"
             break
         fi
     fi
 done <<< "$all_paths"
 
 [ -z "$station_path" ] && { echo "No Station interface found"; exit 1; }
 
 if ! BUS call "$DBUS_SERVICE" "$station_path" net.connman.iwd.Station Scan >/dev/null 2>&1; then
     if ! iwctl_scan; then
         echo "Scan failed (DBus and iwctl)"; exit 1
     fi
 fi
 echo "Scan initiated"
 exit 0
fi

# ============================================================================
# GUI MODE: INITIALIZATION
# ============================================================================

exec 200>"$LOCKFILE"
flock -n 200 || { notify-send "Wi-Fi" "Another instance is already running"; exit 1; }

if [ -z "${DBUS_SESSION_BUS_ADDRESS:-}" ]; then
 notify-send "Wi-Fi Warning" "D-Bus session address not set\n\nScript may not work correctly"
fi

cleanup() {
 if [ -n "${scan_pid:-}" ] && kill -0 "$scan_pid" 2>/dev/null; then
     kill "$scan_pid" 2>/dev/null || true
 fi
 restore_terminal
}
trap 'cleanup; exit 1' ERR
trap cleanup EXIT INT TERM

if ! discover_iwd_paths; then
 notify-send "Wi-Fi Error" "No valid iwd Station/Device found\n\nTry: busctl tree net.connman.iwd"
 exit 1
fi

IFACE=$(get_iface_name)

# ============================================================================
# CLI MODE: JSON OUTPUT
# ============================================================================

if [ -n "${JSON_MODE:-}" ]; then
 print_json_state
 exit 0
fi

if [ -n "$SELF_TEST" ]; then
 self_test
 exit $?
fi

# ============================================================================
# GUI MODE: INITIAL SCAN
# ============================================================================

if [ "$(dbus_get_device_powered)" = "true" ]; then
 dbus_scan_networks
 sleep 2
fi

# ============================================================================
# GUI MODE: MAIN INTERACTION LOOP
# ============================================================================

while true; do
 wifi_status=$(dbus_get_device_powered)
 current_path=$(dbus_get_connected_network_path)
 
 if [ -n "${BASH_XTRACEFD:-}" ]; then
     debug_log "=== DEBUG SNAPSHOT $(date) ==="
     debug_log "State: $(dbus_get_station_state || true)"
     debug_log "Connected: $(dbus_get_connected_network_path || true)"
 fi
 
 if [ "$wifi_status" = "true" ]; then
     control_buttons="[Scan]\n[Known Networks ➜]"
 else
     control_buttons="[WiFi On]"
 fi
 
 if [ "$wifi_status" = "false" ]; then
     chosen=$(printf '%b' "$control_buttons" | rofi_pick "WiFi")
     
     if [ "$chosen" = "[WiFi On]" ]; then
         dbus_set_device_powered true
         notify-send "Wi-Fi" "WiFi enabled"
         sleep 1
         continue
     fi
     continue
 fi
 
 network_list=$(dbus_get_networks || true)
 
 if [ -z "$network_list" ]; then
     dbus_scan_networks
     
     for attempt in {1..5}; do
         sleep 1
         network_list=$(dbus_get_networks || true)
         [ -n "$network_list" ] && break
     done
     
     if [ -z "$network_list" ]; then
         notify-send "Wi-Fi" "No networks found. Try scanning manually." -t 3000
         check_rfkill_status || true
     fi
 fi
 
 menu="$control_buttons"
 declare -A index_to_data
 menu_index=0
 
 while IFS='|' read -r net_path ssid signal_percent sec_type; do
     [ -z "$net_path" ] && continue
     
     icon=$(get_signal_icon "$signal_percent")
     sec_display=$(normalize_security "$sec_type")
     safe_ssid=$(pango_safe "$ssid")
     
     tags=""
     if [ "$net_path" = "$current_path" ]; then
         tags="$tags <b>(connected)</b>"
     fi
     
     if is_known_network "$ssid" "$sec_type" "$IFACE"; then
         tags="$tags [saved]"
     fi
     
     display_line="[$menu_index] $icon $safe_ssid ($sec_display)$tags"
     menu="${menu}\n${display_line}"
     index_to_data["$menu_index"]="$net_path|$ssid|$sec_type"
     menu_index=$((menu_index + 1))
 done <<< "$network_list"
 
 menu="$menu\n[Hidden Network]\n[WiFi Off]"
 
 chosen=$(printf '%b' "$menu" | rofi_pick "WiFi")
 [ -z "$chosen" ] && exit 0
 
 # ========================================================================
 # MENU HANDLER: SCAN
 # ========================================================================
 
 if [ "$chosen" = "[Scan]" ]; then
     (
         while true; do
             dbus_scan_networks
             sleep 3
         done
     ) &
     scan_pid=$!
     
     result=$(printf '%b' "[Stop Scanning]" | rofi_pick "WiFi - Scanning...")
     
     kill $scan_pid 2>/dev/null || true
     scan_pid=""
     
     continue
 fi
 
 # ========================================================================
 # MENU HANDLER: KNOWN NETWORKS
 # ========================================================================
 
 if [ "$chosen" = "[Known Networks ➜]" ]; then
     known_menu=""
     declare -A known_index_to_data
     known_index=0
     
     while IFS='|' read -r known_ssid known_type known_device known_timestamp; do
         [ -z "$known_ssid" ] && continue
         [ "$known_device" != "$IFACE" ] && continue
         
         known_path=$(find_network_by_ssid "$known_ssid" "$network_list" || true)
         known_kn_path="$(find_known_network_path_by_name "$known_ssid" 2>/dev/null || true)"
   
         if [ -z "$known_path" ]; then
             known_display="[$known_index] $(pango_safe "$known_ssid") (out of range)"
             known_menu="${known_menu}${known_display}\n"
             known_index_to_data["$known_index"]="$known_ssid|$known_type|$known_device||$known_kn_path"
             known_index=$((known_index + 1))
             continue
         fi
         
         known_display="[$known_index] $(pango_safe "$known_ssid")"
         if [ "$known_path" = "$current_path" ]; then
             known_display="$known_display <b>(connected)</b>"
         fi
         
         known_menu="${known_menu}${known_display}\n"
         known_index_to_data["$known_index"]="$known_ssid|$known_type|$known_device|$known_path|$known_kn_path"
         known_index=$((known_index + 1))
     done < <(grep -v '^[[:space:]]*$' "$KNOWN_FILE" 2>/dev/null)
     
     known_menu=$(printf '%b' "$known_menu" | sed '/^$/d')
     
     if [ -z "$known_menu" ]; then
         notify-send "Wi-Fi" "No known networks yet\n\nNetworks are saved automatically when you connect"
         continue
     fi
     
     known_chosen=$(printf '%b' "$known_menu" | rofi_pick "Known Networks")
     [ -z "$known_chosen" ] && continue
     
     if [[ "$known_chosen" =~ ^\[([0-9]+)\] ]]; then
         known_idx="${BASH_REMATCH[1]}"
         IFS='|' read -r known_ssid known_type known_device known_path known_kn_path <<< "${known_index_to_data[$known_idx]:-}"
     else
         continue
     fi
     
     safe_prompt=$(pango_safe "$known_ssid")
     
     in_range=false
     if [ -n "$known_path" ]; then
         in_range=true
     fi
     
     if [ "$in_range" = true ]; then
         if [ "$known_path" = "$current_path" ]; then
             known_action=$(printf '%b' "View Details\nDisconnect\nForget Network\nRe-enter Password\nCancel" | rofi_pick "$safe_prompt")
         else
             known_action=$(printf '%b' "Connect\nView Details\nForget Network\nRe-enter Password\nCancel" | rofi_pick "$safe_prompt")
         fi
     else
         known_action=$(printf '%b' "View Details\nForget Network\nCancel" | rofi_pick "$safe_prompt (out of range)")
     fi
     
     case "$known_action" in
         "View Details")
             if [ -n "$known_path" ]; then
                 show_network_details "$known_path" "$known_ssid"
             elif [ -n "$known_kn_path" ]; then
                 show_network_details "$known_kn_path" "$known_ssid"
             else
                 show_network_details "" "$known_ssid"
             fi
             continue
             ;;
         "Disconnect")
             dbus_disconnect
             notify-send "Wi-Fi" "Disconnected from $(safe_label "$known_ssid")"
             continue
             ;;
         "Connect")
             if [ -n "$known_path" ]; then
                 dbus_connect_network "$known_path" ""
                 wait_for_connection "$known_path" "$known_ssid"
             fi
             continue
             ;;
         "Forget Network")
             remove_known_network "$known_ssid" "$known_type" "$known_device"
             forget_network_by_ssid "$known_ssid"
             clear_history "$known_path"
             notify-send "Wi-Fi" "Forgot network $(safe_label "$known_ssid")"
             continue
             ;;
         "Re-enter Password")
             new_pass=$(printf '' | rofi -dmenu -password -p "New password for $safe_prompt")
             if [ -n "$new_pass" ]; then
                 forget_network_by_ssid "$known_ssid"
                 sleep 1
                 
                 if [ -n "$known_path" ]; then
                     dbus_connect_network "$known_path" "$new_pass"
                     wait_for_connection "$known_path" "$known_ssid"
                 else
                     notify-send "Wi-Fi" "Network out of range - password will be used when available"
                 fi
             fi
             continue
             ;;
         *)
             continue
             ;;
     esac
     continue
 fi
 
 # ========================================================================
 # MENU HANDLER: HIDDEN NETWORK
 # ========================================================================
 
 if [ "$chosen" = "[Hidden Network]" ]; then
     hidden_ssid=$(printf '' | rofi_pick "Enter hidden network SSID")
     [ -z "$hidden_ssid" ] && continue
     
     safe_hidden=$(pango_safe "$hidden_ssid")
     
     hidden_type=$(printf '%b' "psk\nsae\nopen\nowe\n8021x" | rofi_pick "Security for $safe_hidden")
     [ -z "$hidden_type" ] && hidden_type="psk"
     
     hidden_pass=""
     case "$(sec_keyword "$hidden_type")" in
         psk|sae|8021x) hidden_pass=$(printf '' | rofi -dmenu -password -p "Password/Passphrase for $safe_hidden") ;;
     esac
     
     if dbus_connect_hidden_network "$hidden_ssid" "$hidden_pass" "$hidden_type"; then
         wait_for_connection "" "$hidden_ssid" || true
         exit 0
     fi
     
     if command -v iwctl &>/dev/null; then
         if [ -n "$hidden_pass" ]; then
             notify-send "Wi-Fi" "D-Bus connection failed. Hidden network connection requires PolicyKit setup for security."
         else
             if IWCTL station "$IFACE" connect "$hidden_ssid" >/dev/null 2>&1; then
                 sleep 3
                 new_path=$(dbus_get_connected_network_path)
                 [ -n "$new_path" ] && notify-send "Wi-Fi" "Connected to $(safe_label "$hidden_ssid")" || notify-send "Wi-Fi" "Connection may have failed for $(safe_label "$hidden_ssid")"
             else
                 notify-send "Wi-Fi" "Connection failed for $(safe_label "$hidden_ssid")"
             fi
         fi
     else
         notify-send "Wi-Fi Error" "Hidden connect not supported via DBus on this build and iwctl is unavailable."
     fi
     exit 0
 fi
 
 # ========================================================================
 # MENU HANDLER: WIFI OFF
 # ========================================================================
 
 if [ "$chosen" = "[WiFi Off]" ]; then
     dbus_set_device_powered false
     notify-send "Wi-Fi" "WiFi disabled"
     continue
 fi
 
 # ========================================================================
 # MENU HANDLER: NETWORK SELECTION
 # ========================================================================
 
 if [[ "$chosen" =~ ^\[([0-9]+)\] ]]; then
     idx="${BASH_REMATCH[1]}"
     IFS='|' read -r net_path ssid sec_type <<< "${index_to_data[$idx]}"
 else
     continue
 fi
 
 if [ -z "$net_path" ]; then
     continue
 fi
 
 if [ "$net_path" = "$current_path" ] && is_connected_state "$(dbus_get_station_state)"; then
     safe_network_prompt=$(pango_safe "$ssid")
     action=$(printf '%b' "View Details\nDisconnect\nRe-enter Password\nCancel" | rofi_pick "$safe_network_prompt")

     case "$action" in
         "View Details")
             show_network_details "$net_path" "$ssid"
             continue
             ;;
         "Disconnect")
             if dbus_disconnect; then
                 notify-send "Wi-Fi" "Disconnected from $(safe_label "$ssid")"
             else
                 notify-send "Wi-Fi" "Failed to disconnect"
             fi
             continue
             ;;
         "Re-enter Password")
             ;;
         *)
             continue
             ;;
     esac
 fi

 is_connected=false
 if [ "$net_path" = "$current_path" ]; then
     is_connected=true
 fi
 
 is_known=false
 if is_known_network "$ssid" "$sec_type" "$IFACE"; then
     is_known=true
 fi
 
 safe_network_prompt=$(pango_safe "$ssid")
 
 if [ "$is_known" = true ] || [ "$is_connected" = true ]; then
     if [ "$is_connected" = true ]; then
         action=$(printf '%b' "View Details\nDisconnect\nForget Network\nRe-enter Password\nCancel" | rofi_pick "$safe_network_prompt")
     else
         action=$(printf '%b' "View Details\nConnect\nForget Network\nRe-enter Password\nCancel" | rofi_pick "$safe_network_prompt")
     fi
     
     case "$action" in
         "View Details")
            show_network_details "$net_path" "$ssid"
            continue
            ;;
         "Disconnect")
             dbus_disconnect
             notify-send "Wi-Fi" "Disconnected from $(safe_label "$ssid")"
             continue
             ;;
         "Connect")
             dbus_connect_network "$net_path" ""
             if ! wait_for_connection "$net_path" "$ssid"; then
                 clear_history "$net_path"
                 notify-send "Wi-Fi" "Saved credentials failed"
             fi
             continue
             ;;
         "Forget Network")
             remove_known_network "$ssid" "$sec_type" "$IFACE"
             clear_history "$net_path"
             forget_network_by_ssid "$ssid"
             notify-send "Wi-Fi" "Forgot network $(safe_label "$ssid")"
             if list_knownnetwork_paths_for_ssid "$ssid" | read -r _; then
               notify-send "Wi-Fi" "Some saved entries for $(safe_label "$ssid") still exist (Polkit/permissions?)"
             fi
             continue
             ;;
         "Re-enter Password")
             ;;
         *)
             continue
             ;;
     esac
 fi
 
 if [ "$sec_type" = "open" ]; then
     dbus_connect_network "$net_path" ""
     wait_for_connection "$net_path" "$ssid"
     continue
 fi
 
 pass=$(printf '' | rofi -dmenu -password -p "Password for $safe_network_prompt")
 if [ -n "$pass" ]; then
   if ! attempt_connect_with_repair "$net_path" "$ssid" "$pass"; then
     notify-send "Wi-Fi" "Connection failed"
   fi
 else
   notify-send "Wi-Fi" "No password entered. Connection cancelled."
 fi
 continue
done
   ```
2. Make it executable:

   ```bash
   chmod +x ~/scripts/Wi-Fi.sh
   ```

3. Create a .desktop file:

   ```bash
   nano ~/.local/share/applications/Wi-Fi.desktop
   ```

   Paste:

   ```ini
   [Desktop Entry]
   Type=Application
   Name=Wi-Fi
   Exec=/home/$USER/scripts/Wi-Fi.sh
   Icon=network-wireless
   Terminal=false
   Categories=Network;
   NoDisplay=true
   ```

4. Make it executable:

   ```bash
   chmod +x ~/.local/share/applications/Wi-Fi.desktop
   ```

---

## rofiLaunchScript

1. Ensure your scripts directory exists:

   ```bash
   mkdir -p ~/scripts
   ```

2. Create launcher script:

   ```bash
   nano ~/scripts/rofi-launcher.sh
   ```

   Paste:

   ```bash
   #!/bin/bash
   iface=$(iwctl device list | sed 's/\x1b\[[0-9;]*m//g' | awk '/wlan/ {print $1; exit}')
   [ -n "$iface" ] && iwctl station "$iface" scan &

   chosen=$(printf "Apps\nWi-Fi\nPower\nExit" | rofi -dmenu -p "Select Action")

   case "$chosen" in
     "Apps") rofi -show drun ;;
     "Wi-Fi") ~/scripts/Wi-Fi.sh ;;
     "Power")
       power_choice=$(printf "Shutdown\nReboot\nSuspend\nCancel" | rofi -dmenu -p "Power")
       case "$power_choice" in
         "Shutdown") systemctl poweroff ;;
         "Reboot") systemctl reboot ;;
         "Suspend") systemctl suspend ;;
       esac
       ;;
     "Exit") exit ;;
   esac
   ```

3. Make it executable:

   ```bash
   chmod +x ~/scripts/rofi-launcher.sh
   ```

4. Bind it in Hyprland:

   ```bash
   nano ~/.config/hypr/hyprland.conf
   ```

   Replace:

   ```ini
   bind = SUPER, SPACE, exec, rofi -show drun
   ```

   With:

   ```ini
   bind = SUPER, SPACE, exec, ~/scripts/rofi-launcher.sh
   ```
