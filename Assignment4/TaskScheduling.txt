###########################################
# Desc: script created for Assignment 4 'Task Scheduling and Powershell' of the INET3700 subject with the following tasks:
#
# Create a PowerShell script that will perform the following tasks:
# 1.	Create a Scheduled Task called: “Scheduled Backup” (10 pts)
# 2.	Run the “BackupMyFiles.ps1” script  (3 pts)
# 3.	On a schedule every 12 hours (2 pts)
# 4.	The “BackupMyFiles.ps1” script should take in three parameters (10 pts)
#   a.	Files to backup
#   b.	Destination of the backups
#   c.	The number of backups that must be saved before overwritten
#
# Date: 07 December 2020
#
# Server Operating Systems and Scripting - George Campanis - INET3700
#
# Author: Laercio M da Silva Junior - W0433181.
#
###########################################

$principal=New-ScheduledTaskPrincipal -UserID "SYSTEM"-LogonType ServiceAccount -RunLevel Highest
$Sta=New-ScheduledTaskAction -Execute "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" –Argument '-ExecutionPolicy Bypass -Command "C:\Temp\BackupMyFiles.ps1 -Destination C:\TempBkp\ -BackupDirs C:\Temp1\,C:\Temp2\  -Versions 3; exit $LASTEXITCODE"'
$Stt=New-ScheduledTaskTrigger -Once -At $(get-date) -RepetitionInterval $([timespan]::FromHours("12"))
 
Register-ScheduledTask Scheduled Backup -Action $Sta -Trigger $Stt -Principal $principal