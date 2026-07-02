# Dragonwilds-Server-Ubuntu

**Overview**

This is a step-by-step guide on how to set up, configure, and run a RuneScape: Dragonwilds dedicated server on a headless Ubuntu server.

**Prerequisites**

- Ubuntu server (20.04 or higher recommended)
- Account with sudo privileges
- System specs: 64-bit OS with a minimum of 2GB RAM + 1GB per player (e.g., 8GB recommended for 6 players)
- Basic knowledge of terminal commands
- Your RuneScape: Dragonwilds **Player ID** (Owner ID) — found in-game at the bottom of the Settings menu. The server will not start without it.

> **Caution:** Directory structures and ports may differ based on your hardware configuration, network setup, and game version. Where I flag something below as unverified, confirm it against your own install before relying on it.

---

# Step 1: Update and Upgrade Your System

sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

---

# Step 2: Install Required Dependencies

**Enable multiverse and install SteamCMD**

The `steamcmd` package lives in Ubuntu's `multiverse` repository, not `main`/`universe`. You need to enable it before installing, or the package won't be found:

sudo apt install -y software-properties-common
sudo add-apt-repository multiverse -y
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install -y steamcmd

> **Note:** During installation you'll be prompted to accept the Steam license agreement. Use the Tab key to select "Ok" and press Enter.

**Install Screen (Session Manager)**

Screen lets your game server console keep running in the background after you close your SSH session.

sudo apt install screen -y

**Install OpenSSH Server**

This enables secure remote access to manage your headless server.

sudo apt install openssh-server -y

**Install UFW (Uncomplicated Firewall)**

sudo apt install ufw -y

---

# Step 3: Configure UFW (Uncomplicated Firewall)

Dragonwilds uses UDP for game traffic. Allow the default port:

sudo ufw allow 7777/udp comment "Dragonwilds Game Port"

> **Tip:** For added security, restrict this to a specific IP address or range instead of allowing all.

> **Note:** If you run additional server instances on the same machine, each needs its own port (7778, 7779, etc.).

**Allow SSH Connections Through UFW** (Optional)

sudo ufw allow from any to any port 22 comment "SSH"

> **Tip:** For added security, change "any" to a specific IP address or range.

Set the default rule to deny incoming traffic (Optional)

sudo ufw default deny incoming

**Enable UFW** (UFW will enable on reboot)

sudo ufw enable

Check the UFW status after enabling it:

sudo ufw status

---

# Step 4: Create a Non-Sudo User

Replace *your_username* with the desired username. This account will run and manage the game server files.

sudo adduser your_username

> **Note:** This will prompt you through the setup.

**Reboot the system**

sudo reboot

---

# Step 5: Download and Install the Dragonwilds Server

Log back into your server after the reboot using your newly created non-sudo user via SSH.

**Fetch Server Binaries Using SteamCMD**

The dedicated server is a free, separate Steam product (App ID `4019830`).

steamcmd +force_install_dir /home/your_username/rs_server +login anonymous +app_update 4019830 validate +quit

**Grant Execution Permissions**

chmod +x /home/your_username/rs_server/RSDragonwilds/Binaries/Linux/RSDragonwildsServer.sh

---

# Step 6: Initial Run and Configuration

**Launch for the first time**

Run the startup script once to generate the base config and default world files:

cd /home/your_username/rs_server/RSDragonwilds/Binaries/Linux
./RSDragonwildsServer.sh -log -NewConsole -Port=7777

Once the console logs show a clean startup, stop the process with `Ctrl + C`.

**Locate and edit DedicatedServer.ini**

⚠️ **Unverified — check before trusting:** Sources disagree on the exact folder name here. The official Dragonwilds wiki and XGamingServer's docs both point to `Saved/Config/Linux/DedicatedServer.ini`, while a Steam Community post from someone self-hosting on Linux, and a separate community setup guide, both say `Saved/Config/LinuxServer/DedicatedServer.ini`. After your first run in Step 6, just check which folder actually exists on your system:

find /home/your_username/rs_server/RSDragonwilds/Saved/Config -iname "DedicatedServer.ini"

Then edit whichever path that returns:

nano /home/your_username/rs_server/RSDragonwilds/Saved/Config/Linux/DedicatedServer.ini

**Add or update the following in the config file:**

[/Script/Dominion.DedicatedServerSettings]
OwnerId=YOUR_RUNESCAPE_DRAGONWILDS_PLAYER_ID
ServerName=My Custom Dragonwilds Server
DefaultWorldName=My World
AdminPassword=YourSecretAdminPassword
WorldPassword=
Public=1

> **Verified against the official wiki, XGamingServer's config docs, and a Steam Community post from a self-hosting Linux user — all three independently confirm this section header and these keys:**
>
> - `OwnerId=` — **Required.** Your Player ID from the in-game Settings menu. The server won't start without it.
> - `ServerName=` — Name shown to players browsing servers.
> - `DefaultWorldName=` — Name of the world created on first startup.
> - `AdminPassword=` — Grants Server Management access to anyone who knows it.
> - `WorldPassword=` — Optional; supersedes any password already stored in the world save. Leave blank to allow anyone to join.
> - `Public=1` — Announces your server via Epic Online Services (EOS). Set to `0` to keep it private/direct-connect only.

**Corrected from an earlier draft of this guide:** an earlier version used a `[MandatorySettings]` / `[OptionalSettings]` section layout with a `MaxPlayers=6` key. I could not find that section header or that key confirmed in any official documentation or first-hand server-operator source — it does not match the wiki, XGamingServer's docs, or the Steam Community thread. The 6-player cap appears to be a fixed engine limit rather than something you configure, so `MaxPlayers` has been dropped. Treat that layout as unconfirmed if you see it elsewhere.

> **Important:** Don't edit `DedicatedServer.ini` while the server is running — changes made during runtime get overwritten and lost. Always stop the server first.

---

# Step 7: Create a Startup Script (Optional)

Return to the user's home directory

cd

Create a directory to place your scripts. Change "*name*" to your desired directory name:

mkdir name

Change to the new directory. Change "*name*" to the one you just created:

cd name

Create a script. Change "*name.sh*" to your desired script name.

nano name.sh

Copy and edit the following script:

#!/bin/bash

#set -x     # Uncomment to enable debug output. This will show you each command as it's executed, which can help identify where it fails

# Log file
LOGFILE="/path/to/your/logfile.txt"  # Update with your log file path
DIRPATH="/path/to/your/server"       # Update with the directory containing the server files

# Create the log directory if it doesn't exist
LOGDIR=$(dirname "$LOGFILE")
mkdir -p "$LOGDIR"

# Create the log file if it doesn't exist
touch "$LOGFILE"

# Function to log messages with date/time
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOGFILE"
}

# Update Dragonwilds using steamcmd
{
    log "Updating Dragonwilds..."
    if /usr/games/steamcmd +force_install_dir "$DIRPATH" +login anonymous +app_update 4019830 validate +quit; then
        log "Update completed."
    else
        log "Update failed."
    fi

    # Make sure the server binary is executable (SteamCMD can reset permissions)
    chmod +x "$DIRPATH/RSDragonwilds/Binaries/Linux/RSDragonwildsServer.sh"

    # Start the Dragonwilds server
    log "Starting Dragonwilds server..."
    if /usr/bin/screen -dmS Dragonwilds "$DIRPATH/RSDragonwilds/Binaries/Linux/RSDragonwildsServer.sh" -log -NewConsole -Port=7777 2>> "$LOGFILE"; then
        log "Dragonwilds server started successfully."
    else
        log "Failed to start Dragonwilds server."
        exit 1  # Exit if the server fails to start
    fi
} 2>&1 | tee -a "$LOGFILE"

Make the script executable by the user:

chmod u+x name.sh

> **Note:** SteamCMD's `validate` flag will overwrite anything it doesn't recognize as part of the base install — including your `DedicatedServer.ini` if it's not otherwise excluded. If you find your config keeps resetting after updates, back up the ini and restore it after each `app_update`, or switch to a staging-directory + `rsync` update pattern.

---

# Step 8: Create a Systemd Service (Optional)

Switch to your sudo user that you used at the beginning. Replace "*your_username*" with the actual username.

su your_username

**Create the service file:**

sudo nano /etc/systemd/system/Dragonwilds.service

**Add the following configuration:**

[Unit]
Description=RuneScape Dragonwilds Dedicated Server
After=network.target

[Service]
Type=simple
User=youruser         # Replace with the username you created in Step 4
ExecStart=/path/to/your/executable/startup/script.sh      # Replace with your full script path
RemainAfterExit=yes
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3
StandardOutput=append:/var/log/dragonwilds.log
StandardError=append:/var/log/dragonwilds.log

[Install]
WantedBy=multi-user.target

> **Example**
>
> User=test
>
> ExecStart=/home/test/scripts/name.sh

**Enable and Start the Service**

sudo systemctl daemon-reload
sudo systemctl enable Dragonwilds.service
sudo systemctl start Dragonwilds.service

> **Important:** This systemd service, along with the accompanying script, ensures that your server automatically starts after a reboot and updates itself before launching.

---

# Step 9: Hardening (Optional)

Login with the sudo user and edit the sshd_config file

sudo nano /etc/ssh/sshd_config

Locate the following lines and uncomment them, making the specified edits:

**#LoginGraceTime 2m**

LoginGraceTime 1m

**#PermitRootLogin prohibit-password**

PermitRootLogin no

**#MaxSessions 10**

MaxSessions 4

Reload systemctl & restart sshd.service

sudo systemctl daemon-reload
sudo systemctl restart ssh.service

## Change Who Can Use the Switch User (su) Command

Make a new group for the su command. Replace "*group_name*" with your desired name for the new group.

sudo groupadd group_name

> **Example:** `sudo groupadd restrictedsu`

**Edit who can use the *su* command**

sudo nano /etc/pam.d/su

Edit the following line to restrict su. Replace "*group_name*" with the one you made earlier.

auth       required   pam_wheel.so group=group_name

> **Example:** `auth required pam_wheel.so group=restrictedsu`

---

**Conclusion**

You have successfully set up your RuneScape: Dragonwilds dedicated server! For further customization, refer to the game's official documentation.

**References**

- https://dragonwilds.runescape.com/news/how-to-dedicated-servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers/Linux
- https://xgamingserver.com/docs/runescape-dragonwilds/config-reference
- https://developer.valvesoftware.com/wiki/SteamCMD#Linux
