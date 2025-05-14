# IwctlWithSystemd
Use systemd-networkd with iwd for a lightweight, minimal networking solution without bloated network managers.

1) Install the iwd package, open your terminal and run:
```
pacman -S iwd
```

2) Enable the iwd service to start automatically:
```
systemctl enable iwd
```

3) Make sure systemd is installed (it should already be on most systems):
```
pacman -S systemd
```

4) Enable the systemd-networkd service for network management:
```
systemctl enable systemd-networkd
```

5) Enable systemd-resolved for DNS resolution:
```
systemctl enable systemd-resolved
```

6) Create the network configuration directory:
```
mkdir -p /etc/systemd/network/
```

7) Create a wireless network configuration file:
```
nano /etc/systemd/network/25-wireless.network
```

8) Paste the following configuration:
```
[Match]
Name=wlan0

[Network]
DHCP=yes
```

9) Exit and save changes: Press Ctrl + O to save, then Ctrl + X to exit.

(Note: If your wireless interface isn't named wlan0, check with `ip link` and use your actual interface name)

10) Create the iwd configuration directory and file:
```
mkdir -p /etc/iwd/
nano /etc/iwd/main.conf
```

11) Paste this configuration to make iwd work with systemd-networkd:
```
[General]
EnableNetworkConfiguration=false

[Network]
NameResolvingService=systemd
```

12) Exit and save changes: Press Ctrl + O to save, then Ctrl + X to exit.

14) Set up DNS resolution by creating a symbolic link:
```
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

14) Reboot your system to apply changes. After rebooting, connect to networks with:
```
iwctl
```

**You're done!** Your system now uses a minimal networking stack with no bloat. 
*The reason for use of systemd instead of iwd alone is to allow for vpn and ethernet use.*


<br>


### Tips for using iwctl:

• List wireless devices: 
```
iwctl device list
```

• Scan for networks: 
```
iwctl station <device> scan
```

• List available networks: 
```
iwctl station <device> get-networks
```

• Connect to a network: 
```
iwctl station <device> connect <network-name>
```

Want a graphical way to connect? Check out the iwctl-rofi integration guide below.

# iwctlUsingRofi
Use Rofi to connect to your Wi-Fi network through iwctl, without a bloated network manager.

1) Create your Wi-Fi picker script, open your terminal and run:
```
nano ~/scripts/Wi-Fi.sh
```

2) Paste the following script:
```
#!/bin/bash

# Detect Wi-Fi interface (strip color codes first)
iface=$(iwctl device list | sed 's/\x1b\[[0-9;]*m//g' | awk '/wlan/ {print $1; exit}')

# Scan for networks
iwctl station "$iface" scan

# Get list of available SSIDs
networks=$(iwctl station "$iface" get-networks |
  sed 's/\x1b\[[0-9;]*m//g' |
  grep -v "Available\|Network\|-----" |
  sed 's/>//g' |
  awk '{if(NF>0) print $1}' |
  grep -v "^$")

# Show menu with Rofi
chosen=$(echo "$networks" | rofi -dmenu -p "Select Wi-Fi")

# If a network was selected, prompt for password
if [ -n "$chosen" ]; then
  # Prompt for password using Rofi
  pass=$(echo "" | rofi -dmenu -password -p "Password for $chosen")

  if [ -n "$pass" ]; then
    # Connect with password
    iwctl station "$iface" connect "$chosen" --passphrase "$pass"
    notify-send "Wi-Fi" "Connecting to $chosen..."
  else
    # Try connecting without password
    iwctl station "$iface" connect "$chosen"
    notify-send "Wi-Fi" "Attempting to connect to $chosen without password..."
  fi
fi
```

3) Exit and save changes: Press Ctrl + O to save, then Ctrl + X to exit.

5) Make the script executable, use the terminal and run:
```
chmod +x ~/scripts/Wi-Fi.sh
```

5) Create a .desktop entry, use the terminal and run:
```
nano ~/.local/share/applications/Wi-Fi.desktop
```

6) Paste the following, replace `yourusername` with your actual Linux username:
```
[Desktop Entry]
Type=Application
Name=Wi-Fi
Exec=/home/yourusername/scripts/Wi-Fi.sh
Icon=network-wireless
Terminal=false
Categories=Network;
```

7) (Optional) Hide from menus, show only in Rofi, add this line to the bottom if you don’t want it appearing in app menus:
```
NoDisplay=true
```

8) Exit and save changes: Press Ctrl + O to save, then Ctrl + X to exit.

10) Make the .desktop file executable, use the terminal and run:
```
chmod +x ~/.local/share/applications/Wi-Fi.desktop
```

**You're done!**


<br>


### Tips for rofi:

• Don't use normal rofi, use the wlroots, wayland native one: 
```
yay -S rofi-lbonn-wayland
```

• If you don't have `yay` yet: 
```
sudo pacman -S base-devel git
cd ~
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

Optionally, you can follow the guide below to create a rofi launcher script, as an alternative to making a Wi-Fi.desktop file.

# rofiLaunchScript

1) Make a scripts directory (if it doesn’t exist):
```
mkdir -p ~/scripts
```

2) Create the launcher script:
```
nano ~/scripts/rofi-launcher.sh
```

3) Paste the following code into the file:
```
#!/bin/bash

# Background Wi-Fi scan
iface=$(iwctl device list | sed 's/\x1b\[[0-9;]*m//g' | awk '/wlan/ {print $1; exit}')
[ -n "$iface" ] && iwctl station "$iface" scan &

# Show Rofi menu
chosen=$(printf "Apps\nWi-Fi\nPower\nExit" | rofi -dmenu -p "Select Action")

# Handle selection
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

4) Exit and save changes: Press Ctrl + O to save, then Ctrl + X to exit.

5) Make the script executable:
```
chmod +x ~/scripts/rofi-launcher.sh
```

6) Edit the Hyprland config:
```
nano ~/.config/hypr/hyprland.conf
```

7) Find the existing line, `bind = SUPER, SPACE, exec, rofi -show drun` And replace it with:
```
bind = SUPER, SPACE, exec, ~/scripts/rofi-launcher.sh
```

9) Exit and save changes: Press Ctrl + O to save, then Ctrl + X to exit.

**Done!**
