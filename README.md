![Build Status](https://img.shields.io/badge/Dedicated_Server-Linux-green)
![Dragonwilds](https://img.shields.io/badge/RuneScape:_Dragonwilds-8A2BE2)
![Steam](https://img.shields.io/badge/Steam-8A2BE2)
# Dragonwilds Linux Server Setup Guide

**Overview**

This is a step-by-step guide on how to set up and run a Ubuntu RuneScape: Dragonwilds server.

**Prerequisites**

- Ubuntu server (20.04 or higher recommended)
- Basic knowledge of terminal commands
- A user with sudo privileges
- Your RuneScape: Dragonwilds **Player ID** (OwnerId) — found in-game at the bottom of the Settings Menu. The server will not start without it.

> [!Caution]
> Directory structures may differ based on your specific setup.

# Step 1: Update and Upgrade Your System

    sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y
--------------------------------------------------------------------------------
# Step 2: Install Required Dependencies

    sudo add-apt-repository multiverse -y
    sudo dpkg --add-architecture i386
    sudo apt update

**Install Screen (Session Manager)**

    sudo apt install screen -y

**Install OpenSSH Sever**

This enables secure remote access to your server.

    sudo apt install openssh-server -y

**Install Steamcmd**

    sudo apt install steamcmd -y

**Install UFW (Uncomplicated Firewall)**

    sudo apt install ufw -y
--------------------------------------------------------------------------------
# Step 3: Configure UFW (Uncomplicated Firewall)

Allow all incoming connections to port 7777:

    sudo ufw allow from any proto udp to any port 7777 comment "Dragonwilds Server Port"

> [!TIP]
 For added security, change "any" to a specific IP address or range.

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
    
--------------------------------------------------------------------------------
# Step 4: Create a Non Sudo User

Replace "*your_username*" with the desired username.

    sudo adduser your_username

> [!NOTE]
> This will prompt you through the setup

**Reboot the system**

    sudo reboot

-------------------------------------------------------------------------------
# Step 5: Install Dragonwilds Server

**Log in to your server with the new user account through cmd, PowerShell, PuTTY, etc. Use your preferred terminal emulator.**

**Install Dragonwilds Server Files** Replace "*YOUR_STEAM_USERNAME*" with the Steam username

    steamcmd +force_install_dir /home/your_username/rs_server +login YOUR_STEAM_USERNAME +app_update 4019830 validate +quit

> [!WARNING]
> `+login anonymous` is what Jagex's own documentation recommends, but it has been observed failing in practice for this app ID. Use a real Steam account that owns the free "RuneScape: Dragonwilds - Dedicated Servers"
> product instead — this is the confirmed working method.

> [!IMPORTANT]
> This interactive step is mandatory before setting up the automated script in Step 7. SteamCMD caches the authenticated session locally after this first successful login, so subsequent scripted `+login
> YOUR_STEAM_USERNAME` calls (run unattended via cron or systemd) reuse the cached token instead of prompting for a password or Steam Guard code. If you skip this step, the automated update script will hang waiting for
> input it will never receive.

**Navigate to the Server Directory**

    cd rs_server

**Start the server**

    ./RSDragonwildsServer.sh -log -NewConsole -Port=7777

Stop the server with Ctrl + C.
--------------------------------------------------------------------------------
# Step 6: Configure the Server

**Edit DedicatedServer.ini**

    nano /home/your_username/rs_server/RSDragonwilds/Saved/Config/LinuxServer/DedicatedServer.ini

**Add the following to DedicatedServer.ini**

    [/Script/Dominion.DedicatedServerSettings]
    OwnerId=YOUR_RUNESCAPE_DRAGONWILDS_PLAYER_ID
    ServerName=My Dragonwilds Server
    DefaultWorldName=My World
    AdminPassword=
    WorldPassword=
    Public=1

> [!TIP]
> Edit the following settings as needed:

OwnerId= (required — your in-game Player ID, found at the bottom of the Settings Menu)

ServerName=""

DefaultWorldName=""

AdminPassword="" (optional)

WorldPassword="" (optional)

> [!Important]
> The server will not start without a valid OwnerId.

> [!NOTE]
> `AdminPassword` and `WorldPassword` are stored in plaintext in `DedicatedServer.ini` with default file permissions — anyone with read access to the file can see them. This is an open item, not something this guide currently locks down.

--------------------------------------------------------------------------------
# Step 7: Create a Startup Script (Optional)

Return to the users home directory

    cd

Create a directory to place you scripts. Change the "*name*" with your desired directory name:

    mkdir name

Change to the new directory. Change the "*name*" with the one you just created:

    cd name

Create a script. Change the "*name.sh*" with your desired script name.

    nano dragonwilds.sh

Copy and edit the following script:

    #!/bin/bash
    
    #set -x     # Uncomment to enable debug output.
    
    # Log file configuration
    LOGFILE="/home/your_username/logs/dragonwilds.txt"  # Update with your actual path
    DIRPATH="/home/your_username/rs_server"             # Update with your actual path
    STEAMUSERNAME="YOUR_STEAM_USERNAME"
    
    # Backup target settings
    BACKUP_DIR="/home/your_username/backups"            # Directory where backups live
    TIMESTAMP=$(date '+%Y-%m-%d_%H%M%S')
    
    # Create necessary directories
    LOGDIR=$(dirname "$LOGFILE")
    mkdir -p "$LOGDIR"    
    mkdir -p "$BACKUP_DIR"
    touch "$LOGFILE"
    
    # Function to log messages with date/time
    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOGFILE"
    }
    
    # Run the backup, update, and start block
    {
        # --- STEP 1: EXECUTE 60-DAY ROTATING BACKUP ---
        log "Initiating pre-update system backup..."
        
        if [ -d "$DIRPATH/RSDragonwilds/Saved" ]; then
            # Package both SaveGames and Config folders into a compressed tarball
            tar -czf "$BACKUP_DIR/dragonwilds_backup_$TIMESTAMP.tar.gz" -C "$DIRPATH/RSDragonwilds" Saved/
            
            if [ $? -eq 0 ]; then
                log "Backup successfully created: dragonwilds_backup_$TIMESTAMP.tar.gz"
            else
                log "Warning: Backup compression encountered errors."
            fi
        else
            log "Warning: Save directory not found. Skipping backup step."
        fi
    
        # Enforce the strict 60-day retention rotation policy
        log "Enforcing 60-day backup retention rotation policy..."
        # Finds files in BACKUP_DIR matching the pattern, older than 60 days, and deletes them
        find "$BACKUP_DIR" -name "dragonwilds_backup_*.tar.gz" -type f -mtime +60 -exec rm -f {} \;
        log "Backup rotation check complete."
    
    
        # --- STEP 2: GAME ENGINE SOFTWARE UPDATE ---
        log "Updating Dragonwilds..."
        if /usr/games/steamcmd +force_install_dir "$DIRPATH" +login "$STEAMUSERNAME" +app_update 4019830 validate +quit; then
            log "Update completed successfully."
        else
            log "Critical: Update failed."
        fi
    
    
        # --- STEP 3: START APPLICATION WINDOW ---
        log "Starting Dragonwilds server inside Screen session..."
        
        # Fire up the screen session safely without double-logging conflicts
        /usr/bin/screen -dmS Dragonwilds "$DIRPATH/RSDragonwildsServer.sh" -log -NewConsole -Port=7777
        
        # Small pause to let Screen initialize so systemd can register the fork
        sleep 2
    
        # Check if the screen session actually exists before claiming victory
        if /usr/bin/screen -list | grep -q "Dragonwilds"; then
            log "Dragonwilds server started successfully in background."
        else
            log "Critical Error: Failed to start Dragonwilds screen session."
            exit 1
        fi
    } 2>&1 | tee -a "$LOGFILE"

Make the script executable by the user:

    chmod +x dragonwilds.sh

--------------------------------------------------------------------------------
# Step 8: Create a Systemd Service (Optional)

Switch to your sudo user that you used at the beginning. Replace "*your_username*" with the actual username.

    su your_username

**Create the service file:**

    sudo nano /etc/systemd/system/Dragonwilds.service

**Add the following configuration:**

    [Unit]
    Description=Dragonwilds Dedicated Game Server
    After=network.target
    StartLimitIntervalSec=60
    StartLimitBurst=3
    
    [Service]
    Type=forking
    User=your_server_user
    ExecStart=/path/to/your/fixed_script.sh
    RemainAfterExit=no
    Restart=on-failure
    RestartSec=10
    
    [Install]
    WantedBy=multi-user.target
    
    
> **Example**
> 
> User=test
> 
> ExecStart=/home/test/scripts/dragonwilds.sh

**Enable and Start the Service**

    sudo systemctl daemon-reload
    sudo systemctl enable Dragonwilds.service
    sudo systemctl start Dragonwilds.service

> [!Important]
>  *This systemd service, along with the accompanying script, ensures that your server automatically starts after a reboot and updates itself before launching.*

--------------------------------------------------------------------------------
# Step 9: Hardening (Optional)

Login with the sudo user and edit the sshd_config file

    sudo nano /etc/ssh/sshd_config

Locate the following lines and uncomment them, making the specified edits:

 **#LoginGraceTime 2m**

    LoginGraceTime 1m

 **#PermitRootLogin prohibit-password**

    PermitRootLogin no

 **#MaxSessions 10**

    Max Sessions 4

Reload systemctl & restart sshd.services

    sudo systemctl daemon-reload
    sudo systemctl restart ssh.service

These are some steps you can take to enhance the security of your SSH service.

## Change Who Can Use the Switch User (su) Command

Make a new group for the su command. Replace "*group_name*" with your desired name for the new group.

    sudo groupadd group_name

> **Example:** *sudo groupadd restrictedsu*

**Edit who can use the *su* command**

Edit the *su* config

    sudo nano /etc/pam.d/su

Edit the following line to restrict su. Replace "*group_name*" with the one you made ealier.

    auth       required   pam_wheel.so group=group_name

> **Example:** *auth       required   pam_wheel.so group=restrictedsu*

# Step 10: How to Restore a Server Backup (Optional)

If an update breaks your server or a world file becomes corrupt, you can easily restore one of your automated backups.

1. **Stop the server service completely:**
   ```bash
   sudo systemctl stop Dragonwilds.service
   ```

2. **Navigate to your backups directory and choose a backup file:**
   ```bash
   cd ~/backups
   ls -la
   ```

3. **Extract the chosen backup over your existing files:**
   *(Replace `TIMESTAMP` with the actual date code on your file)*
   ```bash
   tar -xzf dragonwilds_backup_TIMESTAMP.tar.gz -C ~/rs_server/RSDragonwilds/
   ```

4. **Restart the server service:**
   ```bash
   sudo systemctl start Dragonwilds.service
   ```


**Conclusion**

You have successfully set up your Dragonwilds server! For further customization, refer to the game's official documentation.


**References**
- https://developer.valvesoftware.com/wiki/SteamCMD#Linux
- https://dragonwilds.runescape.com/news/how-to-dedicated-servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers/Linux
- https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands
