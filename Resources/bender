#!/bin/bash
 
# Bender
# Written by Chad Nielsen
# Forget Computers, Get Creative!
 
# Version History
# 1.0 - Initial creation of script for use with a companion launch daemon.
# 1.1 - Moved binary and log locations to /usr/local/robotcloud.
# 1.2 - Code improvements and added compatibility with OS X 10.9 Mavericks.
# 2.0 - postgres backup for profilemanager added by Adrian Burgess
# 2.1 - Added --version flag code.
# 2.2 - Changed to straight serveradmin dump instead of xml.
# 2.3 - Yosemite 10.10 compatibility.
# https://gist.github.com/anotherspot/9145864
 
################################## VARIABLE DEFINITIONS ################################# 
#########################################################################################
 
host=$(hostname)
macOS=$(sw_vers | awk '/ProductVersion/{print substr($2,1,5)}' | tr -d ".")
macSN=$(system_profiler SPHardwareDataType | awk '/Serial Number/{print $4}')
date=$(date +%Y-%m-%d-%H%M)
pass=$(system_profiler SPHardwareDataType | awk '/Hardware UUID/{print $3}')
logPath="/usr/local/robotcloud/logs/bender.log"
pipPath="/usr/local/robotcloud/bin/scroobiuspip"
pipTitle="Bender Backup Error on: $macSN"
backupDestination="/Backups/$date"
keepUntil="14"
version="2.3"
versionCheck="$1"
 
##################################### SCRIPT BEGINS ##################################### 
#########################################################################################
 
function CheckVersion {
	# If a version check is passed to monitor, respond with version but do not run.
	if [ "$versionCheck" = "--version" ]; then
		echo "$version"
	elif [ "$versionCheck" != "" ]; then
		echo "Unknown string passed to binary."
	else
		RobotCloudStructure
		LogEvent "===========================[ Bender is Bending... ]==========================="
		CheckOS
		OpenDirectoryBackup
		ServerAdminBackup
		ProfileManagerBackup
		WikiDBBackup
		WikiFilesBackup
		RemoveOldBackups
		LogEvent "==========================[ Bender has completed ]=========================="
		LogEvent ""
	fi
}

function RobotCloudStructure {
	# Ensure the appropriate directories are in place to generate a log and alert.
	mkdir -p /usr/local/robotcloud/{bin,data,logs}
	# Ensure the log file is present.
	if [ ! -e "$logPath" ]; then
		/usr/bin/touch "$logPath"
	fi
	# Create a new backup destination folder with the execution date.
	/bin/mkdir -p "$backupDestination"
}
 
function CheckOS {
	# Set the serveradmin variable according to operating system.
	echo "$macOS"
	if [ "$macOS" -ge "108" ]; then
		if [ "$macOS" -ge "109" ]; then
			SOCKET="/Library/Server/ProfileManager/Config/var/PostgreSQL" 
			WIKISOCKET="/Library/Server/Wiki/PostgresSocket" 
			DATABASE=devicemgr_v2m0 
			WIKIDATABASE=collab 
		else
			SOCKET="/Library/Server/PostgreSQL For Server Services/Socket"
			WIKISOCKET="/Library/Server/PostgreSQL For Server Services/Socket" 
			DATABASE=device_management
			WIKIDATABASE=collab
		fi
		if [ -e /Applications/Server.app/Contents/ServerRoot/usr/sbin/serveradmin ]; then
			serveradmin="/Applications/Server.app/Contents/ServerRoot/usr/sbin/serveradmin"
			SERVERROOT="/Applications/Server.app/Contents/ServerRoot/usr/bin"
			LogEvent "[ check ] The serveradmin binary has been found."
		else
			LogEvent "[ error ] The serveradmin binary could not be found. Exit Code: $?"
			exit 1
		fi
	elif [ -e /usr/sbin/serveradmin ]; then
		serveradmin="/usr/sbin/serveradmin"
		LogEvent "[ check ] The serveradmin binary has been found."
	else
		LogEvent "[ error ] The serveradmin binary could not be found. Exit Code: $?"
		SendAlert
		exit 1
	fi
}
 
function OpenDirectoryBackup {
	# Check to see if Open Directory service is running.
	obackupDestinationatus=$(sudo $serveradmin status dirserv | grep -c "RUNNING")
	if [ $obackupDestinationatus = 1 ]; then
		# Check to see if Open Directory is set to Master.
		odmaster=$(sudo $serveradmin settings dirserv | grep "LDAPServerType" | grep -c "master")
		if [ $odmaster = 1 ]; then
			LogEvent "[ check ] Open Directory is running and is set to master."
			# Ensure the backup directory is present and assign the path as a variable.
			/bin/mkdir -p $backupDestination/OpenDirectory
			# Instruct the serveradmin binary to create a backup.
			$serveradmin command <<-EOC
				dirserv:backupArchiveParams:archivePassword = $pass
				dirserv:backupArchiveParams:archivePath = ${backupDestination}/OpenDirectory/od_backup-${host}-${date}.sparseimage
				dirserv:command = backupArchive
 
				EOC
			# Check to see if there were any errors backing up Open Directory.
			if [ $? == 0 ]; then
				LogEvent "[ backup ] Open Directory successfully backed up."
			else
				LogEvent "[ error ] There was an error attempting to back up Open Directory. Exit Code: $?"
				SendAlert
				exit 1
			fi
		else
			LogEvent "[ check ] Open Directory not set to master. No backup required."
		fi
	else
		LogEvent "[ check ] Open Directory is not running. No backup required."
	fi			
}
 
function ServerAdminBackup () {
	# Ensure the backup directory is present and assign the path as a variable.
	/bin/mkdir -p $backupDestination/ServerAdmin
	# Create a backup of all services, regardless if they are running or not.
	sudo $serveradmin settings all > $backupDestination/ServerAdmin/sa_backup-allservices-$host-$date.backup
	list=$(sudo $serveradmin list)
	for service in $list; do
	 	sudo $serveradmin settings $service > $backupDestination/ServerAdmin/sa_backup-$service-$host-$date.backup
		if [ $? == 0 ]; then
			LogEvent "[ backup ] $service successfully backed up."
		else
			LogEvent "[ error ] Could not back up $service. Exit Code: $?"
			SendAlert
			exit 1
		fi
	done
}
 
function ProfileManagerBackup () {
	# Ensure the backup directory is present and assign the path as a variable.
	/bin/mkdir -p $backupDestination/ProfileManager
	# Create a backup of profilemanager database.
	sudo $SERVERROOT/pg_dump -v -h $SOCKET --format=c --compress=9 --blobs --username=_devicemgr --file=$backupDestination/ProfileManager/device_management-$host-$date.pgdump $DATABASE
		if [ $? == 0 ]; then
			LogEvent "[ backup ] ProfileManager successfully backed up."
		else
			LogEvent "[ error ] Could not back up ProfileManager. Exit Code: $?"
			SendAlert
			exit 1
		fi
}
 
function WikiDBBackup () {
	# Ensure the backup directory is present and assign the path as a variable.
	/bin/mkdir -p $backupDestination/WikiDB
	# Create a backup of Wiki database.
	sudo $SERVERROOT/pg_dump -v -h $WIKISOCKET --format=c --compress=9 --blobs --username=collab --file=$backupDestination/WikiDB/collab-$host-$date.pgdump $WIKIDATABASE
		if [ $? == 0 ]; then
			LogEvent "[ backup ] Wiki Database successfully backed up."
		else
			LogEvent "[ error ] Could not back up Wiki Database. Exit Code: $?"
			SendAlert
			exit 1
		fi
}
 
function WikiFilesBackup () {
	# Ensure the backup directory is present and assign the path as a variable.
	/bin/mkdir -p $backupDestination/WikiFiles
	# Create a backup of Wiki Files.
	sudo rsync -axv /Library/Server/Wiki/FileData $backupDestination/WikiFiles/
		if [ $? == 0 ]; then
			LogEvent "[ backup ] Wiki Files successfully backed up."
		else
			LogEvent "[ error ] Could not back up Wiki Files. Exit Code: $?"
			SendAlert
			exit 1
		fi
	sudo chown -R root:admin $backupDestination/WikiFiles/
	sudo chmod -R 770 $backupDestination/WikiFiles/
}
 
 
 
function LogEvent () {
	# Echo the passed event and then write it to the log.
	echo $1
	echo $(date "+%Y-%m-%d %H:%M:%S: ") ["bender"] $1 >> "$logPath"
}
 
function RemoveOldBackups () {
	# Remove backups that are older than 14 days.
	LogEvent "[ maintenance ] Pruning files in $1 older than 14 days."
	find /Backups -mtime +$keepUntil -exec rm -rf {} \;
}
 
function SendAlert {
	# Copy the contents of the log to a variable for submission.
	pipDescription=$(cat "$logPath")
	# Ensure that scroobiuspip is installed.
	# This alerting binary is used by Forget Computers.
	# You can modify this area to send alerts using your preferred methods.
	if [ ! -f "$pipPath" ]; then
		LogEvent "[ framework ] Installing scroobiuspip..."
		sudo jamf policy -trigger pip
		# If pip is unable to be installed, add this info so it can be seen in the JSS.
		if [ "$?" = "1" ]; then
			LogEvent "[ error ] Intallation failed. No alert sent."
		else
			LogEvent "[ framework ] Installation successful."
		fi
	else
		LogEvent "[ framework ] The scroobiuspip binary has been found."
	fi
	# Send the email to Zendesk.
	LogEvent ""
	"$pipPath" "$pipTitle" "$pipDescription"
	# If the email was unable to send
	if [ "$?" = "1" ]; then
		LogEvent "[ error ] There was an error sending the alert."
	else
		LogEvent "[ framework ] Successfully sent alert."
	fi
}
 
##################################### FUNCTION CALL ##################################### 
#########################################################################################
 
CheckVersion
 
exit 0