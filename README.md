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

**Navigate to the Server Directory**

    cd rs_server/RSDragonwilds/Binaries/Linux

**Start the server**

    ./RSDragonwildsServer.sh -log -NewConsole -Port=7777

Stop the server with Ctrl + C.
--------------------------------------------------------------------------------
# Step 6: Configure the Server

**Edit DedicatedServer.ini**

    nano /home/your_username/rs_server/RSDragonwilds/Saved/Config/Linux/DedicatedServer.ini

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

    nano name.sh

Copy and edit the following script:

    #!/bin/bash

    #set -x     # Uncomment to enable debug output. This will show you each command as it’s executed, which can help identify where it fails

    # Log file
    LOGFILE="/path/to/your/logfile.txt"  # Update with your log file path
    DIRPATH="/path/to/your/rs_server" # Update with your game server installation directory
    STEAMUSERNAME="YOUR_STEAM_USERNAME"

    # Create the log directory if it doesn't exist
    LOGDIR=$(dirname "$LOGFILE")
    mkdir -p "$LOGDIR"Bi    

    # Create the log file if it doesn't exist
    touch "$LOGFILE"

    # Function to log messages with date/time
    log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOGFILE"
    }

    # Update Dragonwilds using steamcmd
    {
            log "Updating Dragonwilds..."
        if /usr/games/steamcmd +force_install_dir "$DIRPATH" +login "$STEAMUSERNAME" +app_update 4019830 validate +quit; then
            log "Update completed."
        else
            log "Update failed."
        fi

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

    chmod u+x dragonwilds.sh

--------------------------------------------------------------------------------
# Step 8: Create a Systemd Service (Optional)

Switch to your sudo user that you used at the beginning. Replace "*your_username*" with the actual username.

    su your_username

**Create the service file:**

    sudo nano /etc/systemd/system/Dragonwilds.service

**Add the following configuration:**

    [Unit]
    Description=Your Application Description
    After=network.target

    [Service]
    Type=simple
    User=youruser         # Replace with the username you created in the beginning
    ExecStart=/path/to/your/executable/startup/script.sh      # Replace with your full script path
    RemainAfterExit=yes
    Restart=on-failure
    RestartSec=5
    StartLimitIntervalSec=60
    StartLimitBurst=3
    StandardOutput=append:/var/log/yourapp.log
    StandardError=append:/var/log/yourapp.log

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

# Change Who Can Use the Switch User (su) Command

Make a new group for the su command. Replace "*group_name*" with your desired name for the new group.

    sudo groupadd group_name

> **Example:** *sudo groupadd restrictedsu*

**Edit who can use the *su* command**

Edit the *su* config

    sudo nano /etc/pam.d/su

Edit the following line to restrict su. Replace "*group_name*" with the one you made ealier.

    auth       required   pam_wheel.so group=group_name

> **Example:** *auth       required   pam_wheel.so group=restrictedsu*

**Conclusion**

You have successfully set up your Dragonwilds server! For further customization, refer to the game's official documentation.


**References**
- https://developer.valvesoftware.com/wiki/SteamCMD#Linux
- https://dragonwilds.runescape.com/news/how-to-dedicated-servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers
- https://dragonwilds.runescape.wiki/w/Dedicated_Servers/Linux
- https://www.digitalocean.com/community/tutorials/ufw-essentials-common-firewall-rules-and-commands
