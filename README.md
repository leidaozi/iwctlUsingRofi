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
