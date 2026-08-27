![Build Status](https://img.shields.io/badge/Dedicated_Server-Linux-green)
![Dragonwilds](https://img.shields.io/badge/Dragonwilds-8A2BE2)
![Steam](https://img.shields.io/badge/Steam-8A2BE2)
# Dragonwilds Linux Server Setup Guide

**Overview**

This is a step-by-step guide on how to set up and run a Ubuntu Dragonwilds dedicated server, including a hardened backup/update/start pipeline, a graceful stop, an automated update checker, and a systemd unit that correctly tracks the real game process.

**Prerequisites**

- Ubuntu server (20.04 or higher recommended)
- Basic knowledge of terminal commands
- A user with sudo privileges

> [!Caution]
> Directory structures may differ based on your specific setup. Paths below assume the server user is `dragons` and the install directory is `/home/dragons/rs_server` - adjust to match your own setup.

> [!Note]
> Dedicated server support for this game is very new (official support launched with a 2026 update, ahead of the game's own 1.0 release later this year) - the ecosystem here is much thinner than for longer-established titles. There is no RCON, no REST API, and no third-party tooling to speak of yet. Expect this guide to need revisiting as the game matures.

--------------------------------------------------------------------------------
# Step 1: Update and Upgrade Your System

    sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y

--------------------------------------------------------------------------------
# Step 2: Install Required Dependencies

    sudo add-apt-repository multiverse -y
    sudo dpkg --add-architecture i386
    sudo apt update

**Install Screen (Session Manager)**

    sudo apt install screen -y

**Install OpenSSH Server**

    sudo apt install openssh-server -y

**Install Steamcmd**

    sudo apt install steamcmd -y

**Install UFW (Uncomplicated Firewall)**

    sudo apt install ufw -y

--------------------------------------------------------------------------------
# Step 3: Configure UFW (Uncomplicated Firewall)

Allow incoming connections to the game port (the documented default is 7777 - adjust if you use a custom port):

    sudo ufw allow from any proto udp to any port 7777 comment "Dragonwilds Server Port"

**Allow SSH Connections Through UFW** (Optional)

    sudo ufw allow from any to any port 22 comment "SSH"

Set the default rule to deny incoming traffic (Optional)

    sudo ufw default deny incoming

**Enable UFW**

    sudo ufw enable
    sudo ufw status

--------------------------------------------------------------------------------
# Step 4: Create a Non Sudo User

    sudo adduser your_username
    sudo reboot

--------------------------------------------------------------------------------
# Step 5: Install Dragonwilds Server

Log in as your new user, then:

    steamcmd +force_install_dir /home/your_username/rs_server +login anonymous +app_update 4019830 validate +quit

--------------------------------------------------------------------------------
# Step 6: First Run and Configuration

Unlike some other dedicated servers, **Dragonwilds has no RCON and no REST API** - the only remote administration surface is an in-game, password-gated menu (Pause Menu > Server Management), which isn't something a script can drive. The officially documented way to stop the server is `Ctrl+C`, which the stop script in Step 8 uses directly.

Start the server manually once to generate its config files:

    cd rs_server
    ./RSDragonwildsServer.sh -log -NewConsole -port=7777

Wait until the console shows the server has finished starting up, then stop it with `Ctrl+C`. This creates `DedicatedServer.ini` and a default "Standard" world save.

**Edit the config file**

    nano rs_server/RSDragonwilds/Saved/Config/Linux/DedicatedServer.ini

> [!Important]
> The server must be **off** while you edit this file - any changes made while it's running are lost, since the running process holds its own in-memory copy and overwrites the file on exit.

--------------------------------------------------------------------------------
# Step 7: Create the Start Script

    cd
    mkdir -p .scripts logs backups .run .flock
    cd .scripts
    nano start_server.sh

Copy and edit the following - update `DIRPATH`, `SERVER_PORT`, `BACKUP_DIR`, and `LOG_DIR` to match your setup:

    #!/bin/bash
    set -o pipefail
    #set -x

    INSTANCE_NAME="Dragonwilds"
    SERVER_PORT="7777"
    DIRPATH="/home/your_username/rs_server"
    STEAMUSERNAME="anonymous"
    BACKUP_DIR="/home/your_username/backups"
    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    LOG_DIR="/home/your_username/logs"
    LOGFILE="$LOG_DIR/dragonwilds_$TIMESTAMP.log"

    # PID tracking (for real process verification / systemd integration)
    PID_DIR="/home/your_username/.run"
    PID_FILE="$PID_DIR/dragonwilds.pid"
    GAME_PROCESS_NAME="RSDragonwildsServer-Linux-Shipping"

    mkdir -p "$LOG_DIR"
    mkdir -p "$BACKUP_DIR"
    mkdir -p "$PID_DIR"

    # --- SELF-LOCK: prevent overlapping invocations from racing the duplicate check ---
    # Without this, two near-simultaneous calls to this script (from systemd,
    # a control panel, cron, or a manual run) could both pass the duplicate
    # check below before either has actually launched anything, resulting in
    # two real game processes running at once.
    LOCK_DIR="/home/your_username/.flock"
    LOCK_FILE="$LOCK_DIR/start_server.lock"
    mkdir -p "$LOCK_DIR"

    exec 200>"$LOCK_FILE"
    if ! flock -n 200; then
        echo "$(date '+%Y-%m-%d %H:%M:%S') - Another instance of start_server.sh is already running (lock held). Exiting cleanly."
        exit 0
    fi

    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
    }

    {
        # --- STEP 0: DUPLICATE SCREEN SESSION PROTECTION ---
        if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
            log "'${INSTANCE_NAME}' is already running in an active Screen session. Nothing to do."
            # Exit 0, not 1: with Type=forking + Restart=always in the systemd
            # unit, a non-zero exit here would be read as a failed start and
            # trigger an endless restart loop against a server that is
            # actually healthy.
            exit 0
        fi

        # --- STEP 1: BACKUP & LOG ROTATION ---
        log "Initiating pre-update system backup..."
        if [ -d "$DIRPATH/RSDragonwilds/Saved" ]; then
            tar -czf "$BACKUP_DIR/dragonwilds_backup_$TIMESTAMP.tar.gz" -C "$DIRPATH/RSDragonwilds" Saved/
            if [ $? -eq 0 ]; then
                log "Backup successfully created: dragonwilds_backup_$TIMESTAMP.tar.gz"
            else
                log "Warning: Backup compression encountered errors."
            fi
        else
            log "Warning: Save directory not found. Skipping backup step."
        fi

        log "Enforcing 60-day backup retention rotation policy..."
        find "$BACKUP_DIR" -name "dragonwilds_backup_*.tar.gz" -type f -mtime +60 -exec rm -f {} \;
        log "Backup rotation check complete."

        log "Enforcing 30-day individual log retention rotation policy..."
        find "$LOG_DIR" -name "dragonwilds_*.log" -type f -mtime +30 -exec rm -f {} \;
        log "Log rotation check complete."

        # --- STEP 2: GAME ENGINE SOFTWARE UPDATE ---
        log "Updating Dragonwilds..."
        if /usr/games/steamcmd +force_install_dir "$DIRPATH" +login "$STEAMUSERNAME" +app_update 4019830 validate +quit; then
            log "Update completed successfully."
        else
            log "Critical Error: SteamCMD core game update failed. Aborting lifecycle to prevent mismatched version errors."
            exit 1
        fi

        # --- STEP 3: START APPLICATION WINDOW ---
        log "Starting Dragonwilds server inside Screen session..."
        /usr/bin/screen -dmS "$INSTANCE_NAME" "$DIRPATH/RSDragonwildsServer.sh" -log -NewConsole -port="$SERVER_PORT"

        sleep 5

        if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
            log "Screen session is alive. Verifying the actual game process next..."
        else
            log "Critical Error: Failed to start Dragonwilds screen session."
            exit 1
        fi

        # --- STEP 3.5: CAPTURE THE ACTUAL GAME PROCESS PID ---
        log "Waiting for $GAME_PROCESS_NAME to appear so we can confirm the server actually started..."
        PID_WAIT_MAX=60
        PID_WAIT_COUNT=0
        GAME_PID=""

        while [ -z "$GAME_PID" ] && [ $PID_WAIT_COUNT -lt $PID_WAIT_MAX ]; do
            GAME_PID=$(pgrep -f "$GAME_PROCESS_NAME" | head -n1)
            if [ -z "$GAME_PID" ]; then
                sleep 1
                ((PID_WAIT_COUNT++))
            fi
        done

        if [ -n "$GAME_PID" ]; then
            echo "$GAME_PID" > "$PID_FILE"
            log "Dragonwilds server started successfully in background (PID $GAME_PID)."
        else
            log "Critical Error: Screen session is up, but $GAME_PROCESS_NAME never appeared after ${PID_WAIT_MAX}s."
            log "Check the game log for the actual failure cause."
            exit 1
        fi
    } 2>&1 | tee -a "$LOGFILE"

Make it executable:

    chmod u+x start_server.sh

--------------------------------------------------------------------------------
# Step 8: Create the Stop Script

Since there's no RCON or API, `Ctrl+C` (the officially documented method from Step 6) is the only graceful shutdown mechanism available - the fallback here is a direct `SIGKILL`, since there's nothing else to try in between.

    nano stop_server.sh

Copy and edit the following:

    #!/bin/bash
    set -o pipefail

    INSTANCE_NAME="Dragonwilds"
    GAME_PROCESS_NAME="RSDragonwildsServer-Linux-Shipping"

    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    LOG_DIR="/home/your_username/logs"
    LOGFILE="$LOG_DIR/dragonwilds_stop_$TIMESTAMP.log"

    PID_DIR="/home/your_username/.run"
    PID_FILE="$PID_DIR/dragonwilds.pid"

    mkdir -p "$LOG_DIR"

    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
    }

    {
        find "$LOG_DIR" -name "dragonwilds_stop_*.log" -type f -mtime +30 -exec rm -f {} \;
        log "Stop log rotation check complete."

        if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
            log "Active ${INSTANCE_NAME} session discovered. Sending graceful Ctrl+C (the game's own documented stop method)..."

            # Injects a physical Ctrl+C (0x03) into the console, same as the
            # official wiki's own documented shutdown procedure for this game.
            /usr/bin/screen -S "$INSTANCE_NAME" -X stuff $'\003'

            log "Waiting for the world to finish saving and the process to exit..."
            MAX_WAIT=60
            COUNT=0
            while pgrep -f "$GAME_PROCESS_NAME" > /dev/null && [ $COUNT -lt $MAX_WAIT ]; do
                sleep 2
                ((COUNT++))
            done

            SERVER_PID=$(pgrep -f "$GAME_PROCESS_NAME")
            if [ -n "$SERVER_PID" ]; then
                log "Process still alive after Ctrl+C (${MAX_WAIT}s wait, no RCON/API exists for this game to fall back on). Force killing as last resort (possible data loss)."
                kill -9 "$SERVER_PID"
            else
                log "Process exited cleanly after Ctrl+C."
            fi

            if /usr/bin/screen -list | grep -q "\.${INSTANCE_NAME}[[:space:]]"; then
                log "Screen session still present. Force closing."
                /usr/bin/screen -S "$INSTANCE_NAME" -X quit
            fi

            if [ -f "$PID_FILE" ]; then
                rm -f "$PID_FILE"
                log "Removed stale PID file."
            fi

            log "Dragonwilds engine instance safely terminated."
        else
            log "No active server screen found. Lifecycle step skipped."
        fi
    } 2>&1 | tee -a "$LOGFILE"

Make it executable:

    chmod u+x stop_server.sh

--------------------------------------------------------------------------------
# Step 9: Create an Automated Update Checker (Optional)

    nano update_checker.sh

Copy and edit the following, updating paths and `WEBHOOK_URL` (optional) to match your setup:

    #!/bin/bash

    set -o pipefail

    GAME_NAME="Dragonwilds"
    STEAM_APP_ID="4019830"
    SERVER_DIR="/home/your_username/rs_server"
    STEAMCMD="/usr/games/steamcmd"
    STOP_SCRIPT="/home/your_username/.scripts/stop_server.sh"
    START_SCRIPT="/home/your_username/.scripts/start_server.sh"
    SCREEN_NAME="Dragonwilds"
    RESTART_POLL_MAX_WAIT=120
    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    LOG_DIR="/home/your_username/logs"
    LOG_FILE="dragonwilds_update_$TIMESTAMP.log"
    WEBHOOK_URL=""
    IMAGE_URL=""
    DISCORD_FOOTER="Server Maintenance Automation"

    VERSION_FILE="$SERVER_DIR/current_version.txt"
    MANIFEST_FILE="$SERVER_DIR/steamapps/appmanifest_${STEAM_APP_ID}.acf"
    FULL_LOG_PATH="$LOG_DIR/$LOG_FILE"

    mkdir -p "$LOG_DIR"

    log() {
        echo "$1"
    }

    send_discord_message() {
        local message="$1"
        local json_payload
        local emoji="🛠️"

        if [[ "$message" == *"No updates found"* ]]; then
            emoji="🛠️ ℹ️"
        elif [[ "$message" == *"Maintenance Complete"* ]]; then
            emoji="🛠️ 🔄"
        elif [[ "$message" == *"Updates Found"* ]]; then
            emoji="🛠️ ✅"
        elif [[ "$message" == *"Maintenance Failed"* ]]; then
            emoji="🛠️ ❌"
        fi

        if [[ "$message" == *"Maintenance Started"* ]]; then
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "image": { "url": "$IMAGE_URL" },
            "footer": { "text": "$DISCORD_FOOTER" }
        }]
    }
    EOF
    )
        else
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "footer": { "text": "$DISCORD_FOOTER" }
        }]
    }
    EOF
    )
        fi

        if [[ -n "$WEBHOOK_URL" ]]; then
            curl -s -H "Content-Type: application/json" -X POST -d "$json_payload" "$WEBHOOK_URL" > /dev/null 2>&1
        fi
    }

    {
        # Adjust the mtime value below to match your actual cron cadence.
        find "$LOG_DIR" -name "dragonwilds_update_*.log" -type f -mtime +3 -exec rm -f {} \;
        log "$(date '+%Y-%m-%d %H:%M:%S') - Update-checker log rotation check complete."

        if [ -f "$MANIFEST_FILE" ]; then
            REAL_INSTALLED_VERSION=$(grep '"buildid"' "$MANIFEST_FILE" | awk -F '"' '{print $4}' | tr -d '[:space:]')
        else
            log "$(date '+%Y-%m-%d %H:%M:%S') - CRITICAL ERROR: Steam manifest missing at $MANIFEST_FILE."
            exit 1
        fi

        if [ ! -f "$VERSION_FILE" ] || [ ! -s "$VERSION_FILE" ]; then
            echo "$REAL_INSTALLED_VERSION" > "$VERSION_FILE"
        fi

        LOCAL_VERSION=$(cat "$VERSION_FILE" | tr -d '[:space:]')
        if [ "$LOCAL_VERSION" != "$REAL_INSTALLED_VERSION" ]; then
            echo "$REAL_INSTALLED_VERSION" > "$VERSION_FILE"
            LOCAL_VERSION="$REAL_INSTALLED_VERSION"
        fi

        LATEST_VERSION=$("$STEAMCMD" +login anonymous +app_info_update 1 +app_info_print "$STEAM_APP_ID" +quit | \
            awk '/"public"/ {flag=1; next} /}/ && flag {flag=0} flag' | \
            grep '"buildid"' | awk -F '"' '{print $4}' | tr -d '[:space:]')

        if [ $? -ne 0 ] || [ -z "$LATEST_VERSION" ]; then
            log "$(date '+%Y-%m-%d %H:%M:%S') - Error: SteamCMD API query failed. Skipping check."
            exit 1
        fi

        if [ "$LATEST_VERSION" -gt "$LOCAL_VERSION" ]; then
            send_discord_message "Maintenance Started: Updates Found on SteamCMD for $GAME_NAME. Initializing patching pipeline."
            log "$(date +'%Y-%m-%d %H:%M:%S') Executing established graceful shutdown script..."
            bash "$STOP_SCRIPT"

            echo "$LATEST_VERSION" > "$VERSION_FILE"
            log "$(date +'%Y-%m-%d %H:%M:%S') Version file updated to Build ID: $LATEST_VERSION."
            log "$(date +'%Y-%m-%d %H:%M:%S') Server process terminated. Waiting for systemd auto-restart to bring it back..."

            POLL_COUNT=0
            while ! screen -list | grep -q "\.${SCREEN_NAME}[[:space:]]" && [ $POLL_COUNT -lt $RESTART_POLL_MAX_WAIT ]; do
                sleep 2
                ((POLL_COUNT++))
            done

            if screen -list | grep -q "\.${SCREEN_NAME}[[:space:]]"; then
                log "$(date +'%Y-%m-%d %H:%M:%S') Systemd auto-restart succeeded — server back online after $((POLL_COUNT*2))s."
            else
                log "$(date +'%Y-%m-%d %H:%M:%S') WARNING: No screen session detected after $((RESTART_POLL_MAX_WAIT*2))s. Falling back to manual start..."
                send_discord_message "Maintenance Failed: Systemd auto-restart did not bring $GAME_NAME back — falling back to manual start."
                bash "$START_SCRIPT"
                sleep 5
                if screen -list | grep -q "\.${SCREEN_NAME}[[:space:]]"; then
                    log "$(date +'%Y-%m-%d %H:%M:%S') Manual fallback start succeeded — server is back online."
                else
                    log "$(date +'%Y-%m-%d %H:%M:%S') CRITICAL ERROR: Manual fallback also failed. Server is likely DOWN."
                    send_discord_message "Maintenance Failed: Manual fallback start also failed. Server is likely DOWN — manual intervention required."
                    exit 1
                fi
            fi
        else
            log "$(date '+%Y-%m-%d %H:%M:%S') - $GAME_NAME engine fully optimized and up to date."
        fi
    } 2>&1 | tee -a "$FULL_LOG_PATH"

Make it executable, then schedule it (every 30 minutes below):

    chmod u+x update_checker.sh
    crontab -e

Add:

    */30 * * * * flock -n /home/your_username/.flock/update_checker.lock /home/your_username/.scripts/update_checker.sh

--------------------------------------------------------------------------------
# Step 10: Create a Systemd Service

    sudo nano /etc/systemd/system/Dragonwilds.service

**Add the following configuration** - replace `your_username` throughout:

    [Unit]
    Description=Your Dragonwilds Dedicated Server
    After=network.target network-online.target
    Wants=network-online.target
    StartLimitIntervalSec=60
    StartLimitBurst=3

    [Service]
    Type=forking
    User=your_username
    WorkingDirectory=/home/your_username
    ExecStart=/home/your_username/.scripts/start_server.sh
    ExecStop=/home/your_username/.scripts/stop_server.sh
    PIDFile=/home/your_username/.run/dragonwilds.pid
    KillMode=control-group
    SendSIGKILL=no
    TimeoutStartSec=600
    TimeoutStopSec=120
    RemainAfterExit=no
    Restart=always
    RestartSec=60
    StandardOutput=null
    StandardError=null

    [Install]
    WantedBy=multi-user.target

> [!Important]
> - **`Restart=always`, not `Restart=on-failure`**: `on-failure` does not treat a clean `SIGINT`-terminated exit as a failure - which is exactly how this game's own documented `Ctrl+C` shutdown, and this stop script's SIGKILL fallback, terminate the process. Under `on-failure`, a normal graceful stop would silently never trigger systemd's own auto-restart. This was found and fixed on a sibling server in the same infrastructure via a real, reproduced incident.
> - **`TimeoutStopSec=120`**: this game's stop script has no player-facing countdown (no RCON/API to broadcast warnings through), so its worst-case runtime is much shorter than a countdown-based stop script - roughly 60-65s. 120s gives comfortable margin without being wastefully large.
> - **`TimeoutStartSec=600`**: an untested placeholder, not measured runtime the way `TimeoutStopSec` is here - tighten it once you've watched a real update cycle complete and know your actual timing.
> - **`StartLimitIntervalSec=60` / `StartLimitBurst=3`**: caps systemd to 3 restart attempts per minute before giving up and marking the unit `failed`, rather than retrying forever if something is genuinely broken.

**Enable and Start the Service**

    sudo systemctl daemon-reload
    sudo systemctl enable Dragonwilds.service
    sudo systemctl start Dragonwilds.service
    sudo systemctl status Dragonwilds.service
    cat /home/your_username/.run/dragonwilds.pid

--------------------------------------------------------------------------------
# Step 11: Create the Host-Level Maintenance Script (Optional)

Steps 7-10 handle the *game* lifecycle. This step handles the *host*: applying OS security patches and rebooting on a schedule, while cleanly stopping and restarting the game around it. This needs **root**, so log in as your sudo user for this step.

    sudo nano /usr/local/sbin/dragonwilds_maintenance.sh

Copy and edit the following - update `SERVICE_NAME` and `WEBHOOK_URL` (optional) to match your setup:

    #!/bin/bash

    PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

    LOGFILE="/var/log/dragonwilds_system_maintenance.log"
    SERVICE_NAME="Dragonwilds.service"
    WEBHOOK_URL=""
    IMAGE_URL=""

    send_discord_message() {
        local message="$1"
        local json_payload
        local emoji="🛠️"

        if [[ "$message" == *"No updates found"* ]]; then
            emoji="🛠️ ℹ️"
        elif [[ "$message" == *"Maintenance Complete"* ]]; then
            emoji="🛠️ 🔄"
        elif [[ "$message" == *"Updates Found"* ]]; then
            emoji="🛠️ ✅"
        fi

        if [[ "$message" == *"Maintenance Started"* ]]; then
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "image": { "url": "$IMAGE_URL" },
            "footer": { "text": "System Maintenance Automation" }
        }]
    }
    EOF
    )
        else
            json_payload=$(cat <<EOF
    {
        "embeds": [{
            "title": "$emoji $message",
            "color": 16711680,
            "footer": { "text": "System Maintenance Automation" }
        }]
    }
    EOF
    )
        fi

        if [[ -n "$WEBHOOK_URL" ]]; then
            curl -s -H "Content-Type: application/json" -X POST -d "$json_payload" "$WEBHOOK_URL" > /dev/null 2>&1
        fi
    }

    exec > >(tee -a "$LOGFILE") 2>&1

    echo "=============================================================================="
    echo "$(date +'%Y-%m-%d %H:%M:%S') --- Starting Dragonwilds Maintenance Sequence ---"

    # The systemd unit's ExecStop= points at stop_server.sh, which handles
    # the Ctrl+C injection and the SIGKILL fallback. Calling systemctl stop
    # here means that logic lives in exactly one place instead of being
    # duplicated across scripts.
    send_discord_message "Maintenance Started: Stopping the Dragonwilds server via systemd..."
    echo "$(date +'%Y-%m-%d %H:%M:%S') Stopping $SERVICE_NAME (delegates to stop_server.sh for graceful shutdown)..."
    if systemctl stop "$SERVICE_NAME"; then
        echo "$(date +'%Y-%m-%d %H:%M:%S') $SERVICE_NAME stopped successfully."
    else
        echo "$(date +'%Y-%m-%d %H:%M:%S') WARNING: systemctl stop reported a non-zero exit for $SERVICE_NAME. Continuing anyway — check 'systemctl status $SERVICE_NAME' and stop_server.sh's own log if this is unexpected."
    fi

    echo "$(date +'%Y-%m-%d %H:%M:%S') Checking for OS updates..."
    apt-get update > /dev/null

    if apt-get upgrade -s 2>/dev/null | grep -q "^Inst"; then
        send_discord_message "Updates Found. Applying system patches..."
        echo "$(date +'%Y-%m-%d %H:%M:%S') Applying system upgrades..."
        DEBIAN_FRONTEND=noninteractive apt-get -o Dpkg::Options::="--force-confold" -y full-upgrade
        apt-get autoremove -y
        echo "$(date +'%Y-%m-%d %H:%M:%S') OS updates applied successfully."
    else
        send_discord_message "No updates found. System is clean."
        echo "$(date +'%Y-%m-%d %H:%M:%S') No OS updates found."
    fi

    send_discord_message "Maintenance Complete: Rebooting server host. Dragonwilds will auto-update and start on launch."
    echo "$(date +'%Y-%m-%d %H:%M:%S') Maintenance finished. Flushing storage buffers and executing system reboot."
    echo "=============================================================================="

    sync
    sleep 10
    reboot

Make it executable:

    sudo chmod +x /usr/local/sbin/dragonwilds_maintenance.sh

> [!Note]
> `apt-get upgrade -s | grep "^Inst"` (a dry-run simulation) is used deliberately instead of `apt-get list --upgradable` - `apt-get` has no `list` subcommand at all, and that combination silently fails and always reports "no updates," even when real updates are pending.

**Schedule it via root's crontab** (not your game user's):

    sudo crontab -e

Add (5 AM daily shown):

    0 5 * * * flock -n /tmp/dragonwilds_maintenance.lock /usr/local/sbin/dragonwilds_maintenance.sh

--------------------------------------------------------------------------------
# Step 12: Hardening (Optional)

Login with the sudo user and edit the sshd_config file

    sudo nano /etc/ssh/sshd_config

Locate and edit:

    LoginGraceTime 1m
    PermitRootLogin no
    MaxSessions 4

Reload and restart

    sudo systemctl daemon-reload
    sudo systemctl restart ssh.service

**Change Who Can Use the Switch User (su) Command**

    sudo groupadd restrictedsu
    sudo nano /etc/pam.d/su

Add the line:

    auth       required   pam_wheel.so group=restrictedsu

> [!TIP]
> If you want to trigger `start_server.sh`/`stop_server.sh` remotely (e.g. from a control panel or automation tool) without giving that system a general-purpose shell, consider a forced-command SSH key restricted to exactly one script (`command="/home/your_username/.scripts/start_server.sh",restrict ssh-ed25519 ...` in `authorized_keys`) instead of a normal login key. This limits what a leaked key could ever be used for, even in the worst case.

## Lock Down the Operational Scripts

By default, `start_server.sh`, `stop_server.sh`, and `update_checker.sh` are owned by the same user the game process itself runs as. If the running Dragonwilds binary is ever compromised, that account's write access means an attacker could overwrite these scripts - the next time systemd or cron triggers them, your own automation would run the attacker's payload.

    sudo chown root:your_username /home/your_username/.scripts
    sudo chmod 750 /home/your_username/.scripts
    sudo chown root:your_username /home/your_username/.scripts/*.sh
    sudo chmod 750 /home/your_username/.scripts/*.sh

> [!Important]
> Lock down both the **directory** and the **files**. Locking only the files isn't enough - if the directory itself is still writable, an attacker can delete and recreate a script even without write access to its contents.

## Sandbox the systemd Service

Add the following under `[Service]` in `/etc/systemd/system/Dragonwilds.service` (Step 10):

    NoNewPrivileges=true
    PrivateTmp=true
    ProtectSystem=strict
    ProtectHome=read-only
    ReadWritePaths=/home/your_username/rs_server
    ReadWritePaths=/home/your_username/.run
    ReadWritePaths=/home/your_username/.flock
    ReadWritePaths=/home/your_username/logs
    ReadWritePaths=/home/your_username/backups
    ReadWritePaths=/home/your_username/.local/share/Steam

> [!Caution]
> `ProtectSystem=strict` and `ProtectHome=read-only` make essentially the entire filesystem read-only to this service by default - every path it needs to write to must be listed explicitly, or the write fails silently and something breaks (most likely the SteamCMD update or the backup/log/PID-file writes). If you installed SteamCMD differently, confirm its actual cache path first.

**Test before trusting this in production** - reload, restart, and watch a full update cycle complete successfully before considering this done:

    sudo systemctl daemon-reload
    sudo systemctl restart Dragonwilds.service
    sudo journalctl -u Dragonwilds.service -f
    sudo systemd-analyze security Dragonwilds.service

--------------------------------------------------------------------------------

**Conclusion**

You have successfully set up a hardened Dragonwilds server with automated backups, graceful shutdown, and a systemd service that correctly tracks the real game process.

**References**
- https://developer.valvesoftware.com/wiki/SteamCMD#Linux
- https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands
