# Dragonwilds-Server-Ubuntu

**Overview**

This is a step-by-step guide on how to set up, configure, and run a RuneScape: Dragonwilds dedicated server on a headless Ubuntu server.

**Prerequisites**

- Ubuntu server (20.04 or higher recommended)
- Account with sudo privileges
- System specs: 64-bit OS with a minimum of 2GB RAM + 1GB per player (e.g., 8GB recommended for 6 players)
- Basic knowledge of terminal commands
- Your RuneScape: Dragonwilds **Player ID** (Owner ID) — found in-game at the bottom of the Settings menu. The server will not start without it.

> [!CAUTION]
> Directory structures and ports may differ based on your hardware configuration, network setup, and game version.

---

# Step 1: Update and Upgrade Your System

sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

---

# Step 2: Install Required Dependencies

**Enable multiverse and install SteamCMD**

The `steamcmd` package lives in Ubuntu's `multiverse` repository, not `main`/`universe`.

sudo apt install -y software-properties-common
sudo add-apt-repository multiverse -y
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install -y steamcmd

> [!NOTE]
> During installation you'll be prompted to accept the Steam license agreement. Use the Tab key to select "Ok" and press Enter.

**Install Screen (Session Manager)**

Screen lets your game server console keep running in the background after you close your SSH session.

sudo apt install screen -y

**Install OpenSSH Server**

This enables secure remote access to manage your headless server.

sudo apt install openssh-server -y

**Install UFW (Uncomplicated Firewall)**

sudo apt install ufw -y

**Install netcat (used for process/port checks)**

sudo apt install netcat-openbsd -y

---

# Step 3: Configure UFW (Uncomplicated Firewall)

Dragonwilds uses UDP for game traffic. Allow the default port:

sudo ufw allow 7777/udp comment "Dragonwilds Game Port"

> [!TIP]
> For added security, restrict this to a specific IP address or range instead of allowing all.

> [!NOTE]
> If you run additional server instances on the same machine, each needs its own port (7778, 7779, etc.).

**Allow SSH Connections Through UFW** (Optional)

sudo ufw allow from any to any port 22 comment "SSH"

> [!TIP]
> For added security, change "any" to a specific IP address or range.

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

> [!NOTE]
> This will prompt you through the setup.

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

find /home/your_username/rs_server/RSDragonwilds/Saved/Config -iname "DedicatedServer.ini"

nano /home/your_username/rs_server/RSDragonwilds/Saved/Config/Linux/DedicatedServer.ini

> [!WARNING]
> Sources disagree on the exact folder name — some say `Saved/Config/Linux/`, others say `Saved/Config/LinuxServer/`. Run the `find` command above to confirm the real path on your box, then update the `CONFIG_FILE` variable in the Step 7 script to match.

**Add or update the following in the config file:**

[/Script/Dominion.DedicatedServerSettings]
OwnerId=YOUR_RUNESCAPE_DRAGONWILDS_PLAYER_ID
ServerName=My Custom Dragonwilds Server
DefaultWorldName=My World
AdminPassword=YourSecretAdminPassword
WorldPassword=
Public=1

- `OwnerId=` — **Required.** Your Player ID from the in-game Settings menu. The server won't start without it.
- `ServerName=` — Name shown to players browsing servers.
- `DefaultWorldName=` — Name of the world created on first startup.
- `AdminPassword=` — Grants Server Management access to anyone who knows it.
- `WorldPassword=` — Optional; supersedes any password already stored in the world save. Leave blank to allow anyone to join.
- `Public=1` — Announces your server via Epic Online Services (EOS). Set to `0` to keep it private/direct-connect only.

> [!IMPORTANT]
> Don't edit `DedicatedServer.ini` while the server is running — changes made during runtime get overwritten and lost. Always stop the server first.

---

# Step 7: Maintenance Script (Backup, Update, Restore, Restart)

> [!WARNING]
> Unlike 7 Days to Die's Telnet interface, Dragonwilds has no documented RCON/Telnet console. There is no known way to broadcast an in-game warning or issue a remote `saveworld` command before shutdown. This script substitutes a clean `systemctl stop` (which sends SIGTERM and waits up to `TimeoutStopSec`) plus a process-exit verification loop — it cannot warn players in-chat the way your 7DTD script does.

Create the script directory and file:

mkdir -p /home/your_username/scripts
nano /home/your_username/scripts/dragonwilds_maintenance.sh

Copy and edit the following:

#!/bin/bash

LOGFILE="/var/log/dragonwilds_maintenance.log"
SERVICE_NAME="Dragonwilds.service"
DIRPATH="/home/your_username/rs_server"
CONFIG_FILE="$DIRPATH/RSDragonwilds/Saved/Config/Linux/DedicatedServer.ini"   # Confirm path per Step 6 [!WARNING]
BACKUP_DIR="/home/your_username/config_backups"
WEBHOOK_URL=""

send_discord_message() {
    local message="$1"
    if [[ -n "$WEBHOOK_URL" ]]; then
        curl -H "Content-Type: application/json" -X POST -d "{
            \"embeds\": [{
                \"title\": \"🛠️ $message\",
                \"color\": 16711680,
                \"footer\": { \"text\": \"Dragonwilds Server Automation\" }
            }]
        }" "$WEBHOOK_URL" > /dev/null 2>&1
    fi
}

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOGFILE"
}

mkdir -p "$(dirname "$LOGFILE")"
mkdir -p "$BACKUP_DIR"
touch "$LOGFILE"

log "--- STARTING MAINTENANCE ---"
send_discord_message "Maintenance Started: Stopping Server"

# --- 1. GRACEFUL STOP (no in-game warning available — see [!WARNING] above) ---
if systemctl is-active --quiet "$SERVICE_NAME"; then
    log "Service active. Requesting stop..."
    sudo systemctl stop "$SERVICE_NAME"
else
    log "Service already stopped."
fi

# --- 2. VERIFY PROCESS EXIT ---
MAX_WAIT=60; COUNT=0
while pgrep -f "RSDragonwildsServer" > /dev/null && [ $COUNT -lt $MAX_WAIT ]; do
    sleep 2; ((COUNT++))
done

if pgrep -f "RSDragonwildsServer" > /dev/null; then
    log "WARNING: Process did not exit cleanly after ${MAX_WAIT} checks. Forcing kill."
    pkill -f "RSDragonwildsServer"
    send_discord_message "Warning: Server did not stop cleanly, force-killed."
else
    log "Process exited cleanly."
fi

# --- 3. BACK UP CONFIG BEFORE UPDATE ---
if [ -f "$CONFIG_FILE" ]; then
    BACKUP_FILE="$BACKUP_DIR/DedicatedServer.ini.$(date '+%Y%m%d_%H%M%S').bak"
    cp "$CONFIG_FILE" "$BACKUP_FILE"
    cp "$CONFIG_FILE" "$BACKUP_DIR/DedicatedServer.ini.latest.bak"
    log "Config backed up to $BACKUP_FILE"
else
    log "WARNING: No existing config found at $CONFIG_FILE — skipping pre-update backup."
fi

# --- 4. UPDATE VIA STEAMCMD ---
log "Updating Dragonwilds..."
if /usr/games/steamcmd +force_install_dir "$DIRPATH" +login anonymous +app_update 4019830 validate +quit; then
    log "Update completed."
    send_discord_message "Update completed successfully."
else
    log "Update failed."
    send_discord_message "Update FAILED — check logs."
fi

# --- 5. RESTORE CONFIG ---
if [ -f "$BACKUP_DIR/DedicatedServer.ini.latest.bak" ]; then
    cp "$BACKUP_DIR/DedicatedServer.ini.latest.bak" "$CONFIG_FILE"
    log "Config restored from backup after update."
fi

# --- 6. PERMISSIONS ---
chmod +x "$DIRPATH/RSDragonwilds/Binaries/Linux/RSDragonwildsServer.sh"
chmod 600 "$CONFIG_FILE" 2>/dev/null

# --- 7. RESTART SERVICE ---
log "Restarting service..."
if sudo systemctl start "$SERVICE_NAME"; then
    log "Service started successfully."
    send_discord_message "Maintenance Complete: Server is back online."
else
    log "Service failed to start."
    send_discord_message "Maintenance FAILED: Server did not start — check logs."
fi

log "--- MAINTENANCE COMPLETE ---"

Make it executable:

chmod +x /home/your_username/scripts/dragonwilds_maintenance.sh

> [!TIP]
> Point cron or a systemd timer at this script instead of running the raw `app_update` loop from earlier drafts of this guide — it now handles the stop/backup/update/restore/restart cycle in one call, with Discord visibility at each stage, matching the pattern from your 7 Days to Die script.

> [!NOTE]
> This script assumes `systemctl stop`/`start` on the `Dragonwilds.service` unit from Step 8 below — set that up first, or swap in your own start/stop mechanism if you're not using systemd.

---

# Step 8: Create a Systemd Service

Switch to your sudo user that you used at the beginning. Replace "*your_username*" with the actual username.

su your_username

**Create the service file:**

sudo nano /etc/systemd/system/Dragonwilds.service

**Add the following configuration:**

[Unit]
Description=RuneScape Dragonwilds Dedicated Server
After=network.target

[Service]
Type=forking
User=youruser         # Replace with the username you created in Step 4
WorkingDirectory=/home/your_username/rs_server/RSDragonwilds/Binaries/Linux
ExecStart=/usr/bin/screen -dmS Dragonwilds /home/your_username/rs_server/RSDragonwilds/Binaries/Linux/RSDragonwildsServer.sh -log -NewConsole -Port=7777
ExecStop=/usr/bin/screen -S Dragonwilds -X quit
TimeoutStopSec=30
Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=3
StandardOutput=append:/var/log/dragonwilds.log
StandardError=append:/var/log/dragonwilds.log

[Install]
WantedBy=multi-user.target

> [!NOTE]
> Switched to `Type=forking` with a `screen`-wrapped `ExecStart`/`ExecStop` pair here, since the maintenance script in Step 7 now drives updates via `systemctl stop`/`start` rather than the script itself launching the binary directly.

**Enable and Start the Service**

sudo systemctl daemon-reload
sudo systemctl enable Dragonwilds.service
sudo systemctl start Dragonwilds.service

> [!IMPORTANT]
> Run the Step 7 maintenance script (via cron or manually) to handle updates — don't run `app_update` directly against a live install, since it can overwrite `DedicatedServer.ini`.

---

# Step 9: Hardening

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

> [!TIP]
> Example: `sudo groupadd restrictedsu`

**Edit who can use the *su* command**

sudo nano /etc/pam.d/su

Edit the following line to restrict su. Replace "*group_name*" with the one you made earlier.

auth       required   pam_wheel.so group=group_name

> [!TIP]
> Example: `auth required pam_wheel.so group=restrictedsu`

## Lock Down Config File Permissions

The `AdminPassword` and `WorldPassword` values sit in plaintext. Restrict read access to the service user only:

chmod 600 /home/your_username/rs_server/RSDragonwilds/Saved/Config/Linux/DedicatedServer.ini
chmod 700 /home/your_username/config_backups

---

**Conclusion**

You have successfully set up your RuneScape: Dragonwilds dedicated server, with automated backup/update/restore built into a maintenance script and Discord visibility into every stage. For further customization, refer to the game's official documentation.

**References**

- https://dragonwilds.runescape.com/news/how-to-dedicated-servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers/Linux
- https://xgamingserver.com/docs/runescape-dragonwilds/config-reference
- https://developer.valvesoftware.com/wiki/SteamCMD#Linux
