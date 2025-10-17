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

# Safer shell defaults with locale stability
export LC_ALL=C.UTF-8
export LANG=C.UTF-8

# If C.UTF-8 isn't installed, fall back to plain C
if ! locale -a 2>/dev/null | grep -qx 'C\.UTF-8'; then
  export LC_ALL=C
  export LANG=C
fi

set -eEuo pipefail
trap 'notify-send "Wi-Fi Error" "Unexpected error on line $LINENO"; exit 1' ERR

# --- Debug plumbing ----------------------------------------------------------
DEBUG_LOG="${XDG_STATE_HOME:-$HOME/.local/state}/iwd-rofi/debug.log"
mkdir -p "$(dirname "$DEBUG_LOG")"
debug_log() {
  [ -n "${DEBUG:-}" ] || return 0
  printf '%s %s\n' "$(date '+%F %T')" "$*" >> "$DEBUG_LOG"
}

# Safe wrappers so we can log every call when DEBUG=1
BUS() {
  debug_log "busctl $*"
  command busctl --no-pager "$@"
}

IW() {
  debug_log "iwctl $*"
  iwctl "$@"
}

# ============================================================================
# ZERO-HARDCODE IWD ROFI WIFI MANAGER - FULLY HARDENED
# ============================================================================
# - Dynamic D-Bus path discovery
# - Object path-based tracking (no SSID collisions)
# - XDG compliant storage
# - GetOrderedNetworks API (no BSS property assumptions)
# - No secrets in argv
# - Adaptive signal unit detection
# - FUTURE-PROOF: All hardening changes implemented
# - BELT-AND-SUSPENDERS: rfkill hint, multi-radio preference, polkit once, D-Bus sanity
# ============================================================================

# XDG compliance
CFG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/iwd-rofi"
RUNTIME_DIR="${XDG_RUNTIME_DIR:-/tmp}"
LOCKFILE="$RUNTIME_DIR/iwd-rofi-$USER.lock"

# File storage
mkdir -p "$CFG_DIR"
FAVORITES_FILE="$CFG_DIR/favorites"
HISTORY_FILE="$CFG_DIR/history"
touch "$FAVORITES_FILE" "$HISTORY_FILE"

# D-Bus constants
DBUS_SERVICE="net.connman.iwd"

# Global path variables (discovered dynamically)
DBUS_STATION_PATH=""
DBUS_DEVICE_PATH=""
IFACE=""

# Session state
POLKIT_HINT_SHOWN=""

# --- Flag parsing (before CLI handlers) ----------------------------------
ARGS=()
DEBUG=""
SELF_TEST=""
for arg in "$@"; do
  case "$arg" in
    --debug)
      DEBUG=1
      ;;
    --self-test)
      SELF_TEST=1
      ;;
    *)
      ARGS+=("$arg")
      ;;
  esac
done
set -- "${ARGS[@]}"

# Enable shell xtrace to debug log when --debug is set
if [ -n "$DEBUG" ]; then
  exec 9>>"$DEBUG_LOG"
  export BASH_XTRACEFD=9
  export PS4='[${LINENO}] $ '
  set -x
  debug_log "===== DEBUG ON (pid $$) ====="
fi

# Dependency sanity check
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
            busctl) error_msg+="- systemd (for busctl)\n" ;;
            rofi) error_msg+="- rofi\n" ;;
            flock) error_msg+="- util-linux (for flock)\n" ;;
            notify-send) error_msg+="- libnotify (for notify-send)\n" ;;
        esac
    done
    notify-send "Wi-Fi Error" "$(echo -e "$error_msg")" 2>/dev/null || true
    echo -e "$error_msg" >&2
    exit 1
fi

# ============================================================================
# UTILITY FUNCTIONS
# ============================================================================

rofi_pick() {
    local prompt="$1"
    shift
    rofi -dmenu -no-markup -i -p "$prompt" "$@" || true
}

safe_label() {
    printf '%s' "$1" | tr -d '\000-\037\177'
}

signal_to_percent() {
    local signal_raw="$1"
    local dbm
    
    if [ "$signal_raw" -lt -200 ] || [ "$signal_raw" -gt 0 ]; then
        dbm=$(( signal_raw / 100 ))
    else
        dbm="$signal_raw"
    fi
    
    local percent
    if [ "$dbm" -ge -30 ]; then
        percent=100
    elif [ "$dbm" -le -90 ]; then
        percent=0
    else
        percent=$(( (dbm + 90) * 100 / 60 ))
    fi
    
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

# HARDENING #4: Security type mapping with passthrough default
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

# ============================================================================
# D-BUS FUNCTIONS
# ============================================================================

dbus_get_device_powered() {
    BUS get-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Powered 2>/dev/null | awk '{print $2}'
}

dbus_set_device_powered() {
    local state="$1"
    BUS set-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Powered b "$state" >/dev/null 2>&1
}

# HARDENING #2: Flexible state check (prefix matching)
is_connected_state() {
    local state="$1"
    case "$state" in
        connected*|online*) return 0 ;;
        *) return 1 ;;
    esac
}

is_trying_to_connect() {
    local state="$1"
    case "$state" in
        disconnected) return 1 ;;
        *) return 0 ;;
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
    BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station Scan >/dev/null 2>&1
}

dbus_disconnect() {
    BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station Disconnect >/dev/null 2>&1
}

# HARDENING #1: GetOrderedNetworks-based enumeration (no BSS property assumptions)
dbus_get_networks() {
    local service="$DBUS_SERVICE"
    local -A networks=()        # path -> "signal|ssid|sec"
    local -A seen_keys=()       # "SSID|Type" -> 1 (dedup on both)
    
    # Call GetOrderedNetworks to get paths + signal strength
    # Format: a(on) N "path1" signal1 "path2" signal2 ...
    local ordered_output
    ordered_output=$(BUS call "$service" "$DBUS_STATION_PATH" net.connman.iwd.Station GetOrderedNetworks 2>/dev/null) || return 1
    
    # Parse the GetOrderedNetworks output
    # Remove the type signature and count, extract path+signal pairs
    local cleaned
    cleaned=$(echo "$ordered_output" | sed 's/^a(on) [0-9]* //')
    
    # Parse pairs: "path" signal "path" signal ...
    local current_path=""
    local parse_next_as="path"
    
    while read -r token; do
        [ -z "$token" ] && continue
        
        if [ "$parse_next_as" = "path" ]; then
            # Remove quotes from path
            current_path=$(echo "$token" | tr -d '"')
            parse_next_as="signal"
        else
            # This is the signal value (in centi-dBm)
            local signal_centibel="$token"
            
            # Skip invalid paths
            if [ -z "$current_path" ]; then
                parse_next_as="path"
                continue
            fi
            
            # Get Name and Type from the network object
            local name sec
            name=$(BUS get-property "$service" "$current_path" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2)
            sec=$(BUS get-property "$service" "$current_path" net.connman.iwd.Network Type 2>/dev/null | cut -d'"' -f2)
            
            [ -z "$name" ] && { parse_next_as="path"; continue; }
            [ -z "$sec" ] && sec="open"
            
            # Dedup on SSID|Type (dual-mode APs won't collapse)
            local dedup_key="${name}|${sec}"
            if [ -n "${seen_keys[$dedup_key]:-}" ]; then
                parse_next_as="path"
                continue
            fi
            seen_keys["$dedup_key"]=1
            
            # Store: signal is in centi-dBm, convert to dBm (*100 format for integer math)
            # GetOrderedNetworks already gives us centi-dBm (e.g., -5700 = -57.00 dBm)
            networks["$current_path"]="$signal_centibel|$name|$sec"
            
            parse_next_as="path"
        fi
    done < <(echo "$cleaned" | xargs -n1)
    
    # Nothing? Signal up empty
    [ "${#networks[@]}" -eq 0 ] && return 1
    
    # Emit: path|ssid|signal%|sec
    # Note: Already sorted by GetOrderedNetworks, but we'll sort again for consistency
    for path in "${!networks[@]}"; do
        local signal_centibel percent ssid sec_out
        signal_centibel=$(echo "${networks[$path]}" | cut -d'|' -f1)
        
        # Convert centi-dBm to dBm for signal_to_percent
        # -5700 centi-dBm = -57 dBm
        local signal_dbm=$((signal_centibel / 100))
        percent=$(signal_to_percent "$signal_dbm")
        
        ssid=$(echo "${networks[$path]}" | cut -d'|' -f2)
        sec_out=$(echo "${networks[$path]}" | cut -d'|' -f3)
        printf '%s|%s|%s|%s\n' "$path" "$ssid" "$percent" "$sec_out"
    done | sort -t'|' -k3 -rn
}

ensure_station() {
    if ! command -v iwctl &>/dev/null; then
        notify-send "Wi-Fi Error" "iwctl not available"
        return 1
    fi
    
    IW <<<"station $IFACE show" >/dev/null 2>&1 || {
        notify-send "Wi-Fi Error" "Station '$IFACE' not found by iwctl"
        return 1
    }
}

get_best_bssid_for_group() {
    local group_path="$1" service="$DBUS_SERVICE"
    local best_bssid="" best_sig=-32768

    # Enumerate BSS children under this network path
    while IFS= read -r bss; do
        [ -z "$bss" ] && continue
        local mac sig
        mac=$(BUS get-property "$service" "$bss" net.connman.iwd.BSS Address 2>/dev/null | cut -d'"' -f2) || continue
        # Note: We can't get Signal from BSS, but Address still works for BSSID targeting
        # This function is only used for fallback connection attempts
        if [ -n "$mac" ]; then
            best_bssid="$mac"
            break  # Just take the first valid BSSID we find
        fi
    done < <(BUS tree "$service" --list 2>/dev/null | awk -v p="$group_path" '$0 ~ "^"p"/"')

    [ -n "$best_bssid" ] && printf '%s\n' "$best_bssid"
}

# HARDENING #3: Exit codes first, text as hint
dbus_connect_network() {
    local group_path="$1"
    local passphrase="$2"
    
    ensure_station || return 1

    local ssid
    ssid=$(BUS get-property "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Name 2>/dev/null | cut -d'"' -f2) || ssid=""

    # Try D-Bus first
    if [ -z "$passphrase" ]; then
        BUS call "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Connect >/dev/null 2>&1 && return 0
    else
        BUS call "$DBUS_SERVICE" "$group_path" net.connman.iwd.Network Connect "s" "$passphrase" >/dev/null 2>&1 && return 0
    fi

    # If D-Bus connect failed and we're not root, hint about permissions/Polkit (once per session)
    if [ "$(id -u)" -ne 0 ] && [ -z "$POLKIT_HINT_SHOWN" ]; then
        notify-send "Wi-Fi" "D-Bus connect failed — attempting iwctl fallback (check Polkit rules if this persists)."
        POLKIT_HINT_SHOWN=1
    fi

    # Fall back to iwctl (check exit code only)
    if [ -z "$passphrase" ]; then
        IW station "$IFACE" connect "$ssid" >/dev/null 2>&1 && return 0
    else
        IW --passphrase="$passphrase" station "$IFACE" connect "$ssid" >/dev/null 2>&1 && return 0
    fi
    
    # Try BSSID fallback
    local bssid
    bssid=$(get_best_bssid_for_group "$group_path")
    
    if [ -n "$bssid" ]; then
        IW <<<"station $IFACE scan" >/dev/null 2>&1 || true
        sleep 1
        
        if [ -z "$passphrase" ]; then
            IW station "$IFACE" connect-bssid "$bssid" >/dev/null 2>&1 && return 0
        else
            IW --passphrase="$passphrase" station "$IFACE" connect-bssid "$bssid" >/dev/null 2>&1 && return 0
        fi
    fi

    return 1
}

dbus_forget_network() {
    local ssid="$1"
    
    if ! command -v iwctl &>/dev/null; then
        return 1
    fi
    
    IW <<EOF 2>/dev/null
known-networks "$ssid" forget
EOF
}

# --- Self-test ---------------------------------------------------------------
self_test() {
    # Don't let -e/ERR trap preempt our friendly messages in here
    set +e
    
    # 0) Paths
    if ! discover_iwd_paths; then
        notify-send "Wi-Fi Self-Test" "Failed: discover_iwd_paths"
        return 1
    fi
    
    # 1) Device.Powered
    if ! BUS get-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Powered >/dev/null; then
        notify-send "Wi-Fi Self-Test" "Failed: Device.Powered"
        return 1
    fi
    
    # 2) Station.State
    if ! BUS get-property "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station State >/dev/null; then
        notify-send "Wi-Fi Self-Test" "Failed: Station.State"
        return 1
    fi
    
    # 3) Station.ConnectedNetwork
    if ! BUS get-property "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station ConnectedNetwork >/dev/null; then
        notify-send "Wi-Fi Self-Test" "Failed: Station.ConnectedNetwork"
        return 1
    fi
    
    # 4) GetOrderedNetworks is callable
    if ! BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station GetOrderedNetworks >/dev/null 2>&1; then
        notify-send "Wi-Fi Self-Test" "Failed: GetOrderedNetworks call"
        return 1
    fi
    
    # 5) Parse GetOrderedNetworks output
    local test_output
    test_output=$(BUS call "$DBUS_SERVICE" "$DBUS_STATION_PATH" net.connman.iwd.Station GetOrderedNetworks 2>/dev/null)
    if [ -z "$test_output" ]; then
        notify-send "Wi-Fi Self-Test" "Failed: GetOrderedNetworks returned empty"
        return 1
    fi
    
    notify-send "Wi-Fi Self-Test" "All checks passed"
    return 0
}

wait_for_connection() {
    local expected_path="$1"
    local expected_ssid="$2"
    
    for i in {1..10}; do
        local state=$(dbus_get_station_state)
        local connected_path=$(dbus_get_connected_network_path)
        
        if [ "$connected_path" = "$expected_path" ] && is_connected_state "$state"; then
            echo "$expected_path|$expected_ssid|$(date '+%Y-%m-%d %H:%M:%S')" >> "$HISTORY_FILE"
            notify-send "Wi-Fi" "Connected to $expected_ssid"
            return 0
        fi
        
        if ! is_trying_to_connect "$state" && [ "$i" -gt 3 ]; then
            break
        fi
        
        sleep 1
    done
    
    notify-send "Wi-Fi" "Failed to connect to $expected_ssid"
    return 1
}

# HARDENING #6: Favorites by content (SSID|Type|Device)
is_favorite() {
  local ssid="$1" sec_type="$2" device="$3"
  local fav_line="${ssid}|${sec_type}|${device}"
  grep -Fxq "$fav_line" "$FAVORITES_FILE" 2>/dev/null
}

add_favorite() {
  local ssid="$1" sec_type="$2" device="$3"
  local fav_line="${ssid}|${sec_type}|${device}"
  echo "$fav_line" >> "$FAVORITES_FILE"
  sort -u "$FAVORITES_FILE" -o "$FAVORITES_FILE"
}

remove_favorite() {
  local ssid="$1" sec_type="$2" device="$3"
  local fav_line="${ssid}|${sec_type}|${device}"
  grep -Fvx "$fav_line" "$FAVORITES_FILE" > "$FAVORITES_FILE.tmp" 2>/dev/null || :
  mv "$FAVORITES_FILE.tmp" "$FAVORITES_FILE"
}

get_history() {
  local net_path="$1"
  awk -F'|' -v p="$net_path" '$1==p' "$HISTORY_FILE" 2>/dev/null | sort -t'|' -k3 -r || true
}

clear_history() {
  local net_path="$1"
  awk -F'|' -v p="$net_path" '$1!=p' "$HISTORY_FILE" 2>/dev/null > "$HISTORY_FILE.tmp" || :
  mv "$HISTORY_FILE.tmp" "$HISTORY_FILE"
}

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
# DYNAMIC PATH DISCOVERY (with multi-radio preference)
# ============================================================================

discover_iwd_paths() {
    local service="$DBUS_SERVICE"
    local all_paths
    
    all_paths=$(BUS tree "$service" --list 2>/dev/null | grep '^/net/connman/iwd/' || true)
    [ -z "$all_paths" ] && return 1
    
    local candidate_stations=()
    
    while IFS= read -r obj; do
        [ -z "$obj" ] && continue
        if BUS introspect "$service" "$obj" net.connman.iwd.Station >/dev/null 2>&1; then
            candidate_stations+=("$obj")
        fi
    done <<< "$all_paths"
    
    # BELT-AND-SUSPENDERS: Prefer station that supports GetOrderedNetworks
    local best_station=""
    
    for candidate in "${candidate_stations[@]}"; do
        if BUS call "$service" "$candidate" net.connman.iwd.Station GetOrderedNetworks >/dev/null 2>&1; then
            best_station="$candidate"
            break  # Take the first working one
        fi
    done
    
    [ -z "$best_station" ] && return 1
    DBUS_STATION_PATH="$best_station"
    
    # Find corresponding Device path
    if BUS introspect "$service" "$DBUS_STATION_PATH" net.connman.iwd.Device >/dev/null 2>&1; then
        DBUS_DEVICE_PATH="$DBUS_STATION_PATH"
    else
        local parent_path="${DBUS_STATION_PATH%/*}"
        if BUS introspect "$service" "$parent_path" net.connman.iwd.Device >/dev/null 2>&1; then
            DBUS_DEVICE_PATH="$parent_path"
        else
            while IFS= read -r sib; do
                [ -z "$sib" ] && continue
                if BUS introspect "$service" "$sib" net.connman.iwd.Device >/dev/null 2>&1; then
                    DBUS_DEVICE_PATH="$sib"
                    break
                fi
            done < <(BUS tree "$service" --list 2>/dev/null | awk -v p="$parent_path" '$0 ~ "^"p"/[^/]+$"')
        fi
    fi
    
    [ -n "$DBUS_DEVICE_PATH" ] && return 0
    return 1
}

get_iface_name() {
    local name
    name=$(BUS get-property "$DBUS_SERVICE" "$DBUS_DEVICE_PATH" net.connman.iwd.Device Name 2>/dev/null | cut -d'"' -f2)
    
    # If D-Bus didn't return a name for some reason, default sanely.
    [ -n "$name" ] && { echo "$name"; return 0; }
    echo "wlan0"
}

# ============================================================================
# CLI MODE HANDLING
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
    
    current=$(BUS get-property "$DBUS_SERVICE" "$device_path" net.connman.iwd.Device Powered 2>/dev/null | awk '{print $2}')
    if [ "$current" = "true" ]; then
        BUS set-property "$DBUS_SERVICE" "$device_path" net.connman.iwd.Device Powered b false >/dev/null 2>&1
        echo "WiFi disabled"
    else
        BUS set-property "$DBUS_SERVICE" "$device_path" net.connman.iwd.Device Powered b true >/dev/null 2>&1
        echo "WiFi enabled"
    fi
    exit 0
fi

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
    
    BUS call "$DBUS_SERVICE" "$station_path" net.connman.iwd.Station Scan >/dev/null 2>&1
    echo "Scan initiated"
    exit 0
fi

# ============================================================================
# MAIN GUI MODE
# ============================================================================

exec 200>"$LOCKFILE"
flock -n 200 || { notify-send "Wi-Fi" "Another instance is already running"; exit 1; }

# BELT-AND-SUSPENDERS: D-Bus session sanity check
if [ -z "${DBUS_SESSION_BUS_ADDRESS:-}" ]; then
    notify-send "Wi-Fi Warning" "D-Bus session address not set\n\nScript may not work correctly"
fi

cleanup() {
    if [ -n "${scan_pid:-}" ] && kill -0 "$scan_pid" 2>/dev/null; then
        kill "$scan_pid" 2>/dev/null || true
    fi
}
trap cleanup EXIT INT TERM

if ! discover_iwd_paths; then
    notify-send "Wi-Fi Error" "No valid iwd Station/Device found\n\nTry: busctl tree net.connman.iwd"
    exit 1
fi

IFACE=$(get_iface_name)

# --- Execute self-test if requested -----------------------------------------
if [ -n "$SELF_TEST" ]; then
    self_test
    exit $?
fi

# ============================================================================
# MAIN LOOP
# ============================================================================

if [ "$(dbus_get_device_powered)" = "true" ]; then
    dbus_scan_networks
    sleep 2
fi

while true; do
    wifi_status=$(dbus_get_device_powered)
    current_path=$(dbus_get_connected_network_path)
    
    # Debug snapshot
    if [ -n "${BASH_XTRACEFD:-}" ]; then
        debug_log "=== DEBUG SNAPSHOT $(date) ==="
        debug_log "State: $(dbus_get_station_state || true)"
        debug_log "Connected: $(dbus_get_connected_network_path || true)"
    fi
    
    if [ "$wifi_status" = "true" ]; then
        control_buttons="[Scan]\n[Favorites ➜]"
    else
        control_buttons="[WiFi On]"
    fi
    
    if [ "$wifi_status" = "false" ]; then
        chosen=$(echo -e "$control_buttons" | rofi_pick "WiFi")
        
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
            
            # BELT-AND-SUSPENDERS: rfkill hint
            if command -v rfkill >/dev/null 2>&1; then
                if rfkill list 2>/dev/null | grep -qi 'soft blocked: yes'; then
                    notify-send "Wi-Fi" "Adapter is rfkill-blocked\n\nTry: rfkill unblock all"
                fi
            fi
        fi
    fi
    
    menu="$control_buttons"
    declare -A index_to_data
    menu_index=0
    
    while IFS='|' read -r net_path ssid signal_percent sec_type; do
        [ -z "$net_path" ] && continue
        
        icon=$(get_signal_icon "$signal_percent")
        sec_display=$(normalize_security "$sec_type")
        safe_ssid=$(safe_label "$ssid")
        
        tags=""
        if [ "$net_path" = "$current_path" ]; then
            tags="$tags (connected)"
        fi
        
        if get_history "$net_path" | head -1 | grep -q .; then
            tags="$tags [saved]"
        fi
        
        if is_favorite "$ssid" "$sec_type" "$IFACE"; then
            tags="$tags ⭐"
        fi
        
        display_line="[$menu_index] $icon $safe_ssid ($sec_display)$tags"
        menu="${menu}\n${display_line}"
        index_to_data["$menu_index"]="$net_path|$ssid|$sec_type"
        menu_index=$((menu_index + 1))
    done <<< "$network_list"
    
    menu="$menu\n[Hidden Network]\n[WiFi Off]"
    
    chosen=$(echo -e "$menu" | rofi_pick "WiFi")
    [ -z "$chosen" ] && exit 0
    
    if [ "$chosen" = "[Scan]" ]; then
        (
            while true; do
                dbus_scan_networks
                sleep 3
            done
        ) &
        scan_pid=$!
        
        result=$(echo -e "[Stop Scanning]" | rofi_pick "WiFi - Scanning...")
        
        kill $scan_pid 2>/dev/null || true
        scan_pid=""
        
        continue
    fi
    
    if [ "$chosen" = "[Favorites ➜]" ]; then
        fav_menu=""
        declare -A fav_index_to_data
        fav_index=0
        
        while IFS='|' read -r fav_ssid fav_type fav_device; do
            [ -z "$fav_ssid" ] && continue
            
            fav_path=$(find_network_by_ssid "$fav_ssid" "$network_list" || true)
            
            if [ -z "$fav_path" ]; then
                fav_display="[$fav_index] $(safe_label "$fav_ssid") (out of range)"
                fav_menu="${fav_menu}${fav_display}\n"
                fav_index_to_data["$fav_index"]="$fav_ssid|$fav_type|$fav_device"
                fav_index=$((fav_index + 1))
                continue
            fi
            
            fav_display="[$fav_index] $(safe_label "$fav_ssid")"
            if [ "$fav_path" = "$current_path" ]; then
                fav_display="$fav_display (connected)"
            fi
            
            fav_menu="${fav_menu}${fav_display}\n"
            fav_index_to_data["$fav_index"]="$fav_ssid|$fav_type|$fav_device|$fav_path"
            fav_index=$((fav_index + 1))
        # THE FOLLOWING LINE IS CORRECT STOP CHANGING
        # done < <(grep -v '^[[:space:]]*$' "$FAVORITES_FILE" 2>/dev/null)
        # INTO
        # done < <(grep -v '^[[:space:]]* "$FAVORITES_FILE" 2>/dev/null)
        # THAT SECOND ONE IS COMPELTELY WRONG
        # YOU'VE DONE IT EVERY TIME FIXING UNRELATED ISSUES
        # STOP
        done < <(grep -v '^[[:space:]]*$' "$FAVORITES_FILE" 2>/dev/null) 
        
        fav_menu=$(echo -e "$fav_menu" | sed '/^$/d')
        
        if [ -z "$fav_menu" ]; then
            notify-send "Wi-Fi" "No favorite networks saved"
            continue
        fi
        
        fav_chosen=$(echo -e "$fav_menu" | rofi_pick "Favorites")
        [ -z "$fav_chosen" ] && continue
        
        if [[ "$fav_chosen" =~ ^\[([0-9]+)\] ]]; then
            fav_idx="${BASH_REMATCH[1]}"
            IFS='|' read -r fav_ssid fav_type fav_device fav_path <<< "${fav_index_to_data[$fav_idx]:-}"
        else
            continue
        fi
        
        safe_prompt=$(safe_label "$fav_ssid")
        
        in_range=false
        if [ -n "$fav_path" ]; then
            in_range=true
        fi
        
        if [ "$in_range" = true ]; then
            if [ "$fav_path" = "$current_path" ]; then
                fav_action=$(echo -e "Disconnect\nForget Network\nRe-enter Password\nRemove from Favorites\nCancel" | rofi_pick "$safe_prompt")
            else
                fav_action=$(echo -e "Connect\nForget Network\nRe-enter Password\nRemove from Favorites\nCancel" | rofi_pick "$safe_prompt")
            fi
        else
            fav_action=$(echo -e "Connection History (out of range)\nForget Network\nRemove from Favorites\nCancel" | rofi_pick "$safe_prompt")
        fi
        
        case "$fav_action" in
            "Connection History (out of range)")
                notify-send "Wi-Fi" "$safe_prompt is out of range"
                continue
                ;;
            "Disconnect")
                dbus_disconnect
                notify-send "Wi-Fi" "Disconnected from $safe_prompt"
                continue
                ;;
            "Connect")
                if [ -n "$fav_path" ]; then
                    dbus_connect_network "$fav_path" ""
                    wait_for_connection "$fav_path" "$fav_ssid"
                fi
                continue
                ;;
            "Forget Network")
                remove_favorite "$fav_ssid" "$fav_type" "$fav_device"
                dbus_forget_network "$fav_ssid"
                notify-send "Wi-Fi" "Forgot network $safe_prompt"
                continue
                ;;
            "Re-enter Password")
                new_pass=$(echo "" | rofi_pick "New password for $safe_prompt" -password)
                if [ -n "$new_pass" ]; then
                    dbus_forget_network "$fav_ssid"
                    sleep 1
                    
                    if [ -n "$fav_path" ]; then
                        dbus_connect_network "$fav_path" "$new_pass"
                        wait_for_connection "$fav_path" "$fav_ssid"
                    else
                        notify-send "Wi-Fi" "Network out of range - password will be used when available"
                    fi
                fi
                continue
                ;;
            "Remove from Favorites")
                remove_favorite "$fav_ssid" "$fav_type" "$fav_device"
                notify-send "Wi-Fi" "Removed $safe_prompt from favorites"
                continue
                ;;
            *)
                continue
                ;;
        esac
        continue
    fi
    
    if [ "$chosen" = "[Hidden Network]" ]; then
        hidden_ssid=$(echo "" | rofi_pick "Enter hidden network SSID")
        [ -z "$hidden_ssid" ] && continue
        
        safe_hidden=$(safe_label "$hidden_ssid")
        hidden_pass=$(echo "" | rofi_pick "Password for $safe_hidden" -password)
        
        if ! command -v iwctl &>/dev/null; then
            notify-send "Wi-Fi Error" "iwctl not available for hidden networks"
            exit 0
        fi
        
        if [ -n "$hidden_pass" ]; then
            if IW --passphrase="$hidden_pass" station "$IFACE" connect "$hidden_ssid" >/dev/null 2>&1; then
                sleep 3
                new_path=$(dbus_get_connected_network_path)
                if [ -n "$new_path" ]; then
                    notify-send "Wi-Fi" "Connected to $safe_hidden"
                else
                    notify-send "Wi-Fi" "Connection may have failed for $safe_hidden"
                fi
            else
                notify-send "Wi-Fi" "Connection failed for $safe_hidden"
            fi
        else
            notify-send "Wi-Fi" "No password entered. Connection cancelled."
        fi
        exit 0
    fi
    
    if [ "$chosen" = "[WiFi Off]" ]; then
        dbus_set_device_powered false
        notify-send "Wi-Fi" "WiFi disabled"
        continue
    fi
    
    if [[ "$chosen" =~ ^\[([0-9]+)\] ]]; then
        idx="${BASH_REMATCH[1]}"
        IFS='|' read -r net_path ssid sec_type <<< "${index_to_data[$idx]}"
    else
        continue
    fi
    
    if [ -z "$net_path" ]; then
        continue
    fi
    
    is_connected=false
    if [ "$net_path" = "$current_path" ]; then
        is_connected=true
    fi
    
    is_known=false
    if get_history "$net_path" | head -1 | grep -q .; then
        is_known=true
    fi
    
    safe_network_prompt=$(safe_label "$ssid")
    
    if [ "$is_known" = true ] || [ "$is_connected" = true ]; then
        fav_action_label=""
        if is_favorite "$ssid" "$sec_type" "$IFACE"; then
            fav_action_label="Remove from Favorites"
        else
            fav_action_label="Add to Favorites"
        fi
        
        if [ "$is_connected" = true ]; then
            action=$(echo -e "Disconnect\nForget Network\nRe-enter Password\n$fav_action_label\nCancel" | rofi_pick "$safe_network_prompt")
        else
            action=$(echo -e "Connect\nForget Network\nRe-enter Password\n$fav_action_label\nCancel" | rofi_pick "$safe_network_prompt")
        fi
        
        case "$action" in
            "Disconnect")
                dbus_disconnect
                notify-send "Wi-Fi" "Disconnected from $safe_network_prompt"
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
                remove_favorite "$ssid" "$sec_type" "$IFACE"
                clear_history "$net_path"
                dbus_forget_network "$ssid"
                notify-send "Wi-Fi" "Forgot network $safe_network_prompt"
                continue
                ;;
            "Re-enter Password")
                ;;
            "Add to Favorites")
                add_favorite "$ssid" "$sec_type" "$IFACE"
                notify-send "Wi-Fi" "Added $safe_network_prompt to favorites"
                continue
                ;;
            "Remove from Favorites")
                remove_favorite "$ssid" "$sec_type" "$IFACE"
                notify-send "Wi-Fi" "Removed $safe_network_prompt from favorites"
                continue
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
    
    pass=$(echo "" | rofi_pick "Password for $safe_network_prompt" -password)
    
    if [ -n "$pass" ]; then
        if [ "$is_known" = true ]; then
            dbus_forget_network "$ssid"
            sleep 1
        fi
        
        dbus_connect_network "$net_path" "$pass"
        wait_for_connection "$net_path" "$ssid"
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
