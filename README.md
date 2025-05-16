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
   iface=$(iwctl device list | sed 's/\x1b\[[0-9;]*m//g' | awk '/wlan/ {print $1; exit}')
   iwctl station "$iface" scan
   networks=$(iwctl station "$iface" get-networks |
     sed 's/\x1b\[[0-9;]*m//g' |
     grep -v "Available\|Network\|-----" |
     sed 's/>//g' |
     awk '{if(NF>0) print $1}' |
     grep -v "^$")

   chosen=$(echo "$networks" | rofi -dmenu -p "Select Wi-Fi")

   if [ -n "$chosen" ]; then
     pass=$(echo "" | rofi -dmenu -password -p "Password for $chosen")

     if [ -n "$pass" ]; then
       iwctl station "$iface" connect "$chosen" --passphrase "$pass"
       notify-send "Wi-Fi" "Connecting to $chosen..."
     else
       iwctl station "$iface" connect "$chosen"
       notify-send "Wi-Fi" "Attempting to connect to $chosen without password..."
     fi
   fi
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
