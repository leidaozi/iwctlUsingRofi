# IwctlWithSystemd

1) Install iwd:
```
pacman -S iwd
```

2) Enable iwd service:
```
systemctl enable iwd
```

3) Install systemd:
```
pacman -S systemd
```

4) Enable systemd-networkd service:
```
systemctl enable systemd-networkd
```

5) Also enable systemd-resolved for DNS resolution:
```
systemctl enable systemd-resolved
```

6) Create a basic network configuration file for systemd-networkd:
```
mkdir -p /etc/systemd/network/
```

7) Create a wireless network configuration file:
```
nano /etc/systemd/network/25-wireless.network
```

8) Add this content:
```
[Match]
Name=wlan0

[Network]
DHCP=yes
```

(Note: If your wireless interface isn't named wlan0, you'll need to adjust this. You can check with `ip link` to see the correct name)

9) Configure iwd to work with systemd-networkd by creating/editing this file:
```
mkdir -p /etc/iwd/
nano /etc/iwd/main.conf
```

10) Add this content:
```
[General]
EnableNetworkConfiguration=false

[Network]
NameResolvingService=systemd
```

11) If you need to create symbolic link for systemd-resolved:
```
ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

12) After rebooting into your new system, you'll be able to connect to wireless networks using:
```
iwctl
```

# iwctlUsingRofi
Use Rofi to connect to your Wi-Fi network through iwctl, without a bloated network manager. Lightweight, minimal, and perfect for Hyprland or any Wayland setup.

1) Create your Wi-Fi picker script, open your terminal and run:
```
nano ~/.local/bin/Wi-Fi
```

2) Paste the following script:
```
#!/bin/bash

# Detect Wi-Fi interface (strip color codes first)
iface=$(iwctl device list | sed 's/\x1b\[[0-9;]*m//g' | awk '/wlan/ {>

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
    notify-send "Wi-Fi" "Attempting to connect to $chosen without pas>
  fi
fi
```

3) Exit and save changes:
`Press Ctrl + O to save, then Ctrl + X to exit.`

4) Make the script executable, use the terminal and run:
```
chmod +x ~/.local/bin/Wi-Fi
```

5) Create a .desktop entry, use the terminal and run:
```
nano ~/.local/share/applications/Wi-Fi.desktop
```

6) Paste the following, replace 'yourusername' with your actual Linux username:
```
[Desktop Entry]
Type=Application
Name=Wi-Fi
Exec=/home/yourusername/.local/bin/Wi-Fi
Icon=network-wireless
Terminal=false
Categories=Network;
```

7) (Optional) Hide from menus, show only in Rofi, add this line to the bottom if you don’t want it appearing in app menus:
```
NoDisplay=true
```

8) Exit and save changes:
`Press Ctrl + O to save, then Ctrl + X to exit.`

9) Make the .desktop file executable, use the terminal and run:
```
chmod +x ~/.local/share/applications/Wi-Fi.desktop
```

You're Done!
