# Rancid Installation & Script Automation for ASA Backups  
This project explains how to create a **Rancid user** on the Wazuh server, set up **automated daily backups** of Cisco ASA running configurations using **Rancid Tool**, **Expect scripting**, **Bash scripting**, and **crontab scheduling**.

---

## 🎯 Objective  
To automate the secure backup of Cisco ASA device configurations to a Wazuh server, implement monthly and yearly backup retention policies, and execute easy export of backups to local machines when needed. 

---

## 🔍 Why Automate ASA Config Backups?  
Backing up firewall configurations daily is critical for:
- Disaster recovery readiness
- Compliance requirements (e.g., CJIS retention)
- Preventing manual error or loss of configs
- Centralized, version-controlled configuration management

Automation ensures that backups happen consistently without relying on manual intervention.

---

## 📚 Skills Learned  
- Setting up SSH public key authentication for automated access  
- Writing Expect scripts to interact with ASA CLI  
- Bash scripting for file management and retention enforcement  
- Using `crontab` to schedule daily backups
- Exporting configuration backups to a local machine using `scp` or `pscp`, depending on OS

---

## 🛠️ Tools Used  
<div>
  <a href="https://shrubbery.net/rancid/" target="_blank"><img src="https://img.shields.io/badge/-Rancid-800000?&style=for-the-badge&logoColor=white" />
  <img src="https://img.shields.io/badge/-Expect-333333?&style=for-the-badge&logo=Linux&logoColor=white" />
  <img src="https://img.shields.io/badge/-Bash_Scripting-4EAA25?&style=for-the-badge&logo=GNU-Bash&logoColor=white" />
    <img src="https://img.shields.io/badge/-RHEL-EE0000?&style=for-the-badge&logo=Red-Hat&logoColor=white" />
</div>

---

## 📝 Deployment Steps

### 1. Set Up the Rancid User on Wazuh Server
```bash
sudo useradd -m -s /bin/bash rancid
sudo passwd rancid
sudo usermod -aG wheel rancid
```
Verify group membership:
```bash
groups rancid
```

### 2. Configure SSH Key Authentication
Check to see if you have any keys:
```bash
sudo ls -l ~/.ssh
```
On the rancid user account:
```bash
ssh-keygen -t rsa -b 2048
sudo ls -l ~/.ssh
```

### 3. Now you want to add the public key to the ASA.
- Add a rancid user in users/AAA.
- Then add the newly generated public key to the user for easy ssh access.
- Then run ssh rancid@<SITE-ASA-IP> (run this on your rancid user in the Wazuh box) to try and access the server, you should be able to log right in with no pass since it has a key, this is for automation.

### 4. Access SITE ASA from rancid user
Make sure you can access SITE ASA from rancid user via Public Key, and make sure you can run necessary commands on SITE ASA that we will have Rancid automate and run:
ssh into rancid user on SITE Wazuh box.
```bash
ssh rancid@<Wazuh-Server-IP>
```
Access SITE ASA from rancid user via Public Key.
```bash
ssh rancid@<SITE-ASA-IP>
```
```bash
enable
```
- type in password
```bash
more system:running-config
```
(outputs running config that we need backups daily of) hold space bar to go through it fast.
Make sure to have Expect, Zip and rancid installed on rancid user in Wazuh Box
enable repository and install rancid and dependencies:
```bash
sudo dnf install epel-release -y
sudo dnf install rancid expect -y
```
```bash
sudo yum install zip -y
```
You might need to Download the epel-release RPM manually, if other command doesn't work
```bash
sudo dnf install \
  https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm -y
```

### 5. Make your backup directories
Make your backup directories and give them the proper permissions
my main path is /var/config_backup (so cd here)
subdirectories:
```bash
sudo mkdir /var/config_backup
```
```bash
sudo mkdir /var/config_backup/SITE-monthly
```
```bash
sudo mkdir /var/config_backup/SITE-yearly
```
Make sure you’re in /var directory and run these permission commands
```bash
sudo chown -R rancid:wheel /var/config_backup
```
```bash
sudo chmod -R 775 /var/config_backup
```
```bash
sudo chown -R rancid:wheel /var/config_backup/SITE-monthly
sudo chown -R rancid:wheel /var/config_backup/SITE-yearly
```
```bash
sudo chmod -R 775 /var/config_backup/SITE-monthly
sudo chmod -R 775 /var/config_backup/SITE-yearly
```
```bash
sudo chmod g+s /var/config_backup
```
Then run this to ensure you’re permissions worked
```bash
ls -ld /var/config_backup/SITE-monthly
```
Should output this: drwxrwxr-x. 2 rancid wheel 6 Apr  1 15:48 /var/config_backup/SITE-monthly

### 6. Create the expect script
Create the expect script that will access SITE ASA from rancid user, run necessary commands, and extract the whole running config from SITE ASA to rancid user in Wazuh box:
```bash
/usr/bin/expect <<EOF
set timeout -1
spawn ssh $USER@$HOST
expect {
    ">" { send "enable\r" }
    eof { puts "Error: Unable to establish SSH connection."; exit 1 }
}
# Enter enable mode
expect "Password:"
send "$ENABLE_PASSWORD\r"
expect {
    "#" { send "more system:running-config\r" }
    eof { puts "Error: Failed to enter enable mode."; exit 1 }
}
# Capture the output
set full_output ""
while {1} {
    expect {
        "<--- More --->" {
            send " "
            append full_output \$expect_out(buffer)
        }
        ": end" {
            append full_output \$expect_out(buffer)
            break
        }
        eof {
            append full_output \$expect_out(buffer)
            break
        }
        timeout {
            puts "Error: Script timed out during output retrieval."
            exit 1
        }
    }
}
# Save the output to a file
set file [open "$CONFIG_FILE" w]
puts \$file \$full_output
close \$file
# Exit the session
send "exit\r"
expect eof
EOF
```
This Expect script will be embedded in the main Bash Script

### Secure Your Enable Password
Now to avoid storing sensitive credentials directly in your script, you'll store the enable password securely in a separate environment file with limited read permissions.
Create a file to store your ASA enable password:
```bash
sudo nano /home/rancid/backup_scripts/.backup_env
```
Add this to the file:
```bash
ENABLE_PASSWORD="yourEnablePasswordHere"
```
Save and exit with Ctrl + O, Enter, then Ctrl + X (For nano)
Then Restrict File Access
Run:
```bash
sudo chown rancid:wheel /home/rancid/backup_scripts/.backup_env
sudo chmod 600 /home/rancid/backup_scripts/.backup_env
```
This ensures only the rancid user can access the password.

### 7. Create the rest of the Bash Script
Now create the rest of the Bash Script that will run the rest of the process including retention policy; making the backup directories (monthly and yearly) if not already made; zipping files and removing none zipped files for clean up; copying all backups to daily dir for 30 day retention and every first of the month back up to the yearly dir for yearly retention; deleting files in these dir that are passed their retention period; embed the Expect script into this Bash Script. You can make a dir on your rancid home called “backup_scripts” where this script can live. It’ll be called SITE-ASA-Config-Backups.sh:
```bash
#!/bin/bash

# Load secure environment variables
source /home/rancid/backup_scripts/.backup_env

# Configuration
HOST="<SITE-ASA-IP>"
USER="rancid"
BACKUP_DIR="/var/config_backup"
MONTHLY_DIR="$BACKUP_DIR/SITE-monthly"
YEARLY_DIR="$BACKUP_DIR/SITE-yearly"
TODAY=$(date +"%Y-%m-%d")  # Current date
FIRST_OF_MONTH=$(date -d "$TODAY" +"%Y-%m-01")  # First day of the current month
CURRENT_YEAR=$(date +"%Y")
CONFIG_FILE="$MONTHLY_DIR/config-$TODAY.txt"
ZIP_FILE="$MONTHLY_DIR/config-backup-$TODAY.zip"
# Retention Policy
MONTHLY_RETENTION_DAYS=30
YEARLY_RETENTION_DAYS=365
# Ensure directories exist
# mkdir -p "$MONTHLY_DIR" "$YEARLY_DIR"
# Run the Expect script
/usr/bin/expect <<EOF
set timeout -1
spawn ssh $USER@$HOST
expect {
    ">" { send "enable\r" }
    eof { puts "Error: Unable to establish SSH connection."; exit 1 }
}
# Enter enable mode
expect "Password:"
send "$ENABLE_PASSWORD\r"
expect {
    "#" { send "more system:running-config\r" }
    eof { puts "Error: Failed to enter enable mode."; exit 1 }
}
# Capture the output
set full_output ""
while {1} {
    expect {
        "<--- More --->" {
            send " "
            append full_output \$expect_out(buffer)
        }
        ": end" {
            append full_output \$expect_out(buffer)
            break
        }
        eof {
            append full_output \$expect_out(buffer)
            break
        }
        timeout {
            puts "Error: Script timed out during output retrieval."
            exit 1
        }
    }
}
# Save the output to a file
set file [open "$CONFIG_FILE" w]
puts \$file \$full_output
close \$file
# Exit the session
send "exit\r"
expect eof
EOF
# Zip the backup file
if [[ -f "$CONFIG_FILE" ]]; then
    zip -j "$ZIP_FILE" "$CONFIG_FILE"
    if [[ $? -ne 0 ]]; then
        echo "Error: Failed to create zip file."
        exit 1
    fi
    rm "$CONFIG_FILE"  # Remove the raw .txt file after zipping
else
    echo "Error: Configuration file $CONFIG_FILE not found."
    exit 1
fi
# Move first-of-month backups to the yearly directory
if [[ "$TODAY" == "$FIRST_OF_MONTH" ]]; then
    cp "$ZIP_FILE" "$YEARLY_DIR/config-backup-$TODAY.zip"
    if [[ $? -ne 0 ]]; then
        echo "Error: Failed to copy zip file to yearly directory."
        exit 1
    fi
fi
# Clean up daily backups older than the retention period
find "$MONTHLY_DIR" -type f -mtime +$MONTHLY_RETENTION_DAYS -exec rm {} \;
# Clean up yearly backups older than the retention period
find "$YEARLY_DIR" -type f -mtime +$YEARLY_RETENTION_DAYS -exec rm {} \;
echo "Backup process completed successfully."
```
Save and exit the text editor:
- Vi editor use: esc then :wq!
- Nano editor use: ctrl+o enter and ctrl +x
Give this script executable permissions so we can run it
```bash
chmod +x SITE-ASA-Config-Backups.sh
```
- Run it using ./SITE-ASA-Config-Backups.sh , and you should see the config and a success message.

### 8. Now we can test the script
- Run the script and ensure you get the “Backup process completed successfully” message
- We can check the /var/config_backup/SITE-monthly dir to see if our zip config file is there, and check the /var/config_backup/SITE-yearly dir to make sure the same zip was not copied there since today is most likely not the first of the month
- Now we can test the yearly dir, so change line 10 to show todays date as “2024-01-01”, instead of “%Y-%m-%d” and run this script again. Now check the /var/config_backup/SITE-yearly dir and you should see this new zip in the dir, and you should also see it in the SITE-monthly dir. Make sure to revert the script current date back to original
- Now we can test the removal of zips that are passed their retention date
Run these commands to add 2 files to the daily dir, one being in retention period and another being passed the retention period
```bash
touch -d "35 days ago" /var/config_backup/SITE-monthly/test-monthly-old.zip
touch -d "25 days ago" /var/config_backup/SITE-monthly/test-monthly-recent.zip
```
```bash
ls -l /var/config_backup/SITE-monthly
```
- Now execute the script and list the monthly dir again and you will see that the recent zip is still there but the older zip has been removed
- Now for the yearly retention we can run these commands to add a recent and older file to the yearly dir
```bash
touch -d "370 days ago" /var/config_backup/SITE-yearly/test-yearly-old.zip
touch -d "360 days ago" /var/config_backup/SITE-yearly/test-yearly-recent.zip
```
- List the yearly dir to see the 2 files there
```bash
ls -l /var/config_backup/SITE-yearly
```
Execute the script and list the yearly dir again to see that the older file was removed and the recent one was kept

### 9. Add this script to our crontab
- Run this “crontab -e” to edit the crontab, use “i” to insert text, “esc” to stop inserting, and “:wq!” to save and exit the vi editor, and add this to the crontab
```bash
0 2 * * * /home/rancid/backup_scripts/SITE-ASA-Config-Backups.sh
```
This will automate the script to run daily at 2 am, check the monthly and yearly directories daily to ensure its running properly


### Rancid ASA Config Backups (Export Commands)
- First, ssh into the Wazuh box you need the ASA config backup from via CLI.
```bash
ssh user@<Wazuh-Server-IP>
```
- Next, cd to the directory in which you’d like to export a config backup from
the mother directory “config_backup” holds all the monthly  and yearly directories for each ASA
```bash
cd /var/config_backup
```
Inside this directory you will see 2 SITE directories. The monthly directory holds daily ASA config backups for 30 days then deletes them. The yearly directories hold all first day of the month (meaning 01/01/2025; 02/01/2025; 03/01/2025; etc.) ASA config backups for 365 days then deletes them.
To access the SITE directories for example you can use the cd commands below, and use “ls” to list what is in the directories, or you can cd to /var/config_backup and run “ls -R config_backup” to quickly list all directories, in your current directory, and all files inside them:
```bash
cd /var/config_backup/SITE-monthly 
```
```bash
cd /var/config_backup/SITE-yearly 
```
- Now we can choose a config to export
Lets say I’m on a Mac and I want to export “config-backup-2025-04-01.zip” from SITE-monthly dir to my local desktop, I can run a scp command for this. Make sure you are on your local computer terminal when running this, not ssh’d into any other server. Also make sure to change the user from mtagaras to your user:
```bash
scp mtagaras@<Wazuh-Server-IP>:/var/config_backup/SITE-monthly/config-backup-2025-04-01.zip /Users/mtagaras/Desktop/
```
You should now have a copy of the config.zip file on your desktop. You can change the path that this zip file goes to, like a certain folder, by changing the path at the end of the command (/Users/mtagaras/Desktop/).
If you are on Windows you can use the pscp command, make sure to change the user from mtagaras to your user:
```bash
pscp -scp mtagaras@<Wazuh-Server-IP>:/var/config_backup/SITE-monthly/config-backup-2025-04-01.zip %USERPROFILE%\Desktop
```

---

### 👨‍💻 Author
Mario Tagaras | Florida State University Alum







