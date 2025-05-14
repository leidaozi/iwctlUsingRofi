# iwctlUsingRofi
Use Rofi to connect to your Wi-Fi network through iwctl, without a bloated network manager. Lightweight, minimal, and perfect for Hyprland or any Wayland setup.

1) Create your Wi-Fi picker script, open your terminal and run:
nano ~/.local/bin/Wi-Fi

2) Paste the following script:
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

3) Exit and save changes:
Press Ctrl + O to save, then Ctrl + X to exit.

4) Make the script executable, use the terminal and run:
chmod +x ~/.local/bin/Wi-Fi

5) Create a .desktop entry, use the terminal and run:
nano ~/.local/share/applications/Wi-Fi.desktop

6) Paste the following, replace 'yourusername' with your actual Linux username:
[Desktop Entry]
Type=Application
Name=Wi-Fi
Exec=/home/yourusername/.local/bin/Wi-Fi
Icon=network-wireless
Terminal=false
Categories=Network;

7) (Optional) Hide from menus, show only in Rofi, add this line to the bottom if you don’t want it appearing in app menus:
NoDisplay=true

8) Exit and save changes:
Press Ctrl + O to save, then Ctrl + X to exit.

9) Make the .desktop file executable, use the terminal and run:
chmod +x ~/.local/share/applications/Wi-Fi.desktop

You're Done!

Now you can:

    Launch your script via Rofi drun mode
    
    Keep your system free of bloated Wi-Fi daemons
    
    Manage your own interface with elegant precision
