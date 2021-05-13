###########################################
# Desc: script created for assignment 3 of the INET3700 subject with the following functionalities on Windows Local Accounts:
# 1. Add Windows Local User
# 2. Change a Local User Password
# 3. Add a Local User To a LocalGroup
# 4. Remove a Local User From a LocalGroup
# 5. Remove a Local User from Windows
#
# Date: 20 November 2020
#
# Server Operating Systems and Scripting - George Campanis - INET3700
#
# Author: Laercio M da Silva Junior - W0433181.
#
# FOR A BETTER SCRIPT EXPERIENCE, SELECT ALL THE CODE AND RUN IT ALL AT ONCE
###########################################

# *** Functions used in more than one task. These functions need to be RUN FIRST.***
# function used to take and validate the username entered by the user
function UsernameValidation {
    [CmdletBinding()]
    param (
        [string] $Message
    )

    [bool] $UserValidation = $false

    do {
        $UserName = Read-Host $Message

        if (($UserName -eq "c") -or ($UserName -eq "cancel")) {
            WindowsLocalAccountsSetupScript
        }

        if (!$UserName) {
            Write-Host "`nNo username was entered.`n"
        }
        elseif (@(Get-LocalUser | Where-Object { $_.Name -eq $UserName }).Count -eq 0) {
            Write-Host "`n$UserName does not exist.`n"
        }
        else {
            $UserValidation = $true
        }
    } while ( !$UserValidation )

    return $UserName
}

# function used to take and validate the password entered by the user
function PwValidation {
    [CmdletBinding()]
    param (
        [string] $UserName
    )

    [bool] $PasswordValidation = $false

    do {
        $Pw = Read-Host "`nEnter a new password for $UserName" #-MaskInput # Available in PowerShell Version 7 content

        if (($Pw -eq "c") -or ($Pw -eq "cancel")) {
            WindowsLocalAccountsSetupScript
        }
        elseif (!$Pw) {
            Write-Host "`nNo password was entered.`n"
            continue                      
        }

        $ConfirmationPw = Read-Host "`nEnter the same password again to confirm" #-MaskInput # Available in PowerShell Version 7 content

        if (($ConfirmationPw -eq "c") -or ($ConfirmationPw -eq "cancel")) {
            WindowsLocalAccountsSetupScript
        }
        elseif (!$ConfirmationPw) {
            Write-Host "`nNo password was entered.`n"
            continue
        }

        if ($Pw -eq $ConfirmationPw) {
            $PasswordValidation = $true 
        }
        else {
            Write-Host "`nPasswords are not the same. Please enter the same password.`n"
        }

    } while ( !$PasswordValidation )

    $PwSecure = ConvertTo-SecureString $Pw -asplaintext -force
    return $PwSecure
}

# function used to take and validate the localgroup entered by the user
function LocalgroupValidation {
    [CmdletBinding()]
    param (
        [string] $Message
    )

    [bool] $GroupValidation = $false

    do {
        $Group = Read-Host $Message

        if (($Group -eq "c") -or ($Group -eq "cancel")) {
            WindowsLocalAccountsSetupScript
        }

        if (@(Get-LocalGroup | Where-Object { $_.Name -eq $Group }).Count -eq 0) {
            Write-Host "`n$Group LocalGroup does not exist.`n"
        }
        else {
            $GroupValidation = $true
        }
    } while ( !$GroupValidation )

    return $Group
}




# 1.	Add a Local User (7pts) and create a Function AddWindowsUser with parameters for “UserName”, “Password”, “LocalGroup”
function AddWindowsUser {
    [CmdletBinding()]
    param (
        [string] $UserName,
        [securestring] $Password,
        [string] $LocalGroup
    )
    
    try {
        New-LocalUser -Name $UserName -Password $Password -FullName $UserName -Description "New test User $UserName"
        Write-EventLog -EntryType SuccessAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7001 -Message "Windows User Created"
        Write-Host "`n$UserName was successfully created"
    }
    catch {
        Write-EventLog -EntryType FailureAudit  -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7002 -Message "Windows User was NOT Created"
        Write-Host "`n$UserName was NOT created."
    }

    if ($LocalGroup) {
        try {
            Add-LocalGroupMember -Group $LocalGroup -Member $UserName
            Write-EventLog -EntryType SuccessAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7003 -Message "Windows User Created and Added to a LocalGroup"
            Write-Host "`n$UserName was successfully added to $LocalGroup LocalGroup"
        }
        catch {
            Write-EventLog -EntryType FailureAudit  -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7004 -Message "Windows User Created but was NOT Added to a LocalGroup"
            Write-Host "`n$UserName was NOT added to $LocalGroup LocalGroup."
        }
    }
}

function MenuAddWindowsUser {

    [bool] $UserValidation = $false
    do {
        $UserName = Read-Host "`nEnter the username for which you want to create a new Windows account"

        if (($UserName -eq "c") -or ($UserName -eq "cancel")) {
            WindowsLocalAccountsSetupScript
        }

        if (!$UserName) {
            Write-Host "`nNo username was entered.`n"
        }
        elseif (@(Get-LocalUser | Where-Object { $_.Name -eq $UserName }).Count -eq 0) {
            $UserValidation = $true
        }
        else {
            Write-Host "`nUsername $UserName is already in use. Please try a different username than $UserName`n"
        }
    } while ( !$UserValidation )

    [securestring] $Password = PwValidation -UserName $UserName

    [bool] $GroupValidation = $false
    do {
        $Group = Read-Host "`nEnter the LocalGroup for which you want to add a new Windows account. If you don't want to add the user to a group, just hit enter"

        if (($Group -eq "c") -or ($Group -eq "cancel")) {
            WindowsLocalAccountsSetupScript
        }

        if (!$Group) {
            $GroupValidation = $true
        }
        elseif (@(Get-LocalGroup | Where-Object { $_.Name -eq $Group }).Count -eq 0) {
            Write-Host "`n$Group LocalGroup does not exist.`n"
        }
        else {
            $GroupValidation = $true
        }
    } while ( !$GroupValidation )

    AddWindowsUser -UserName $UserName -Password $Password -LocalGroup $Group -Verbose
}


# 1.1 List all users and groups to check if the user was created and added to correct group
function SecretFunctionOne {
    [CmdletBinding()]
    $adsi = [ADSI]"WinNT://$env:COMPUTERNAME"
    $adsi.Children | Where-Object {$_.SchemaClassName -eq 'user'} | Foreach-Object {
        $groups = $_.Groups() | Foreach-Object {$_.GetType().InvokeMember("Name", 'GetProperty', $null, $_, $null)}
        $_ | Select-Object @{n='UserName';e={$_.Name}},@{n='Groups';e={$groups -join ';'}}
    }
}




# 2.	Change Password (5pts) and create a Function ChangeUserPassword with parameters for “UserName”, “Password”
function ChangeUserPassword {
    [CmdletBinding()]
    param (
        [string] $UserName,
        [securestring] $Password
    )
    
    try {
        Get-LocalUser -Name $UserName | Set-LocalUser -Password $Password
        Write-EventLog -EntryType SuccessAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7005 -Message "Windows User Password Changed"
        Write-Host "`nPassword was successfully changed"
    }
    catch {
        Write-EventLog -EntryType FailureAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7006 -Message "Windows User Password was NOT Changed"
        Write-Host "`nPassword was NOT changed."
    }
}

function MenuChangeUserPassword {
    [string] $UserName = UsernameValidation -Message "`nEnter the username for which you want to change the password"
    [securestring] $Password = PwValidation -UserName $UserName
    ChangeUserPassword $UserName -Password $Password -Verbose
}

# 2.1 Check if the password was changed
function SecretFunctionTwo {
    [string] $UserName = UsernameValidation -Message "`nEnter the username you want to check the password for"
    [string] $Password = Read-Host "`nEnter the password"

    $computer = $env:COMPUTERNAME
    Add-Type -AssemblyName System.DirectoryServices.AccountManagement
    $obj = New-Object System.DirectoryServices.AccountManagement.PrincipalContext('machine',$computer)
    Write-Host $obj.ValidateCredentials($UserName, $Password)
}




# 3.	Add a User to an Existing Local Group (5pts) and create a Function AddToLocalGroup with parameters for “UserName”, “LocalGroup”
function AddToLocalGroup {
    [CmdletBinding()]
    param (
        [string] $UserName,
        [string] $LocalGroup
    )
    
    try {
        Add-LocalGroupMember -Group $LocalGroup -Member $UserName
        Write-EventLog -EntryType SuccessAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7007 -Message "Windows User Added to a LocalGroup"
        Write-Host "`n$UserName was successfully added to $LocalGroup LocalGroup"
    }
    catch {
        Write-EventLog -EntryType FailureAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7008 -Message "Windows User was NOT Added to a LocalGroup"
        Write-Host "`n$UserName was NOT added to $LocalGroup LocalGroup."
    }
}

function MenuAddToLocalGroup {
    [string] $UserName = UsernameValidation -Message "`nEnter the username you want to change the LocalGroup"
    [string] $Group = LocalgroupValidation -Message "`nEnter the LocalGroup for which you want to add $UserName"
    AddToLocalGroup -UserName $UserName -LocalGroup $Group -Verbose
}

# 3.1 Check if the User was added to correct LocalGroup
function SecretFunctionThree {
    $Group = LocalgroupValidation -Message "`nEnter the LocalGroup you want to see all users who belong to it"
    $prompt = Get-LocalGroupMember -Group $Group
    foreach ($item in $prompt) {
        Write-Host $item
    }
}
# Or
# @(Get-LocalGroupMember -Group $Group | ? { $_.Name.Substring($_.Name.IndexOf("\")+1) -eq $UserName }).Count -ne 0




# 4.	Remove a User (5pts) and create a Function RemoveFromLocalGroup with parameters for “UserName”, “LocalGroup” and create a Function RemoveWindowsUser with parameters for “UserName”
function RemoveFromLocalGroup {
    [CmdletBinding()]
    param (
        [string] $UserName,
        [string] $LocalGroup
    )
    
    try {
        Remove-LocalGroupMember -Group $LocalGroup -Member $UserName
        Write-EventLog -EntryType SuccessAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7009 -Message "Windows User Removed from a LocalGroup"
        Write-Host "`n$UserName was successfully removed from $LocalGroup LocalGroup"
    }
    catch {
        Write-EventLog -EntryType FailureAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7010 -Message "Windows User was NOT Removed from a LocalGroup"
        Write-Host "`n$UserName was NOT removed from $LocalGroup LocalGroup."
    }
}

function MenuRemoveFromLocalGroup {
    [string] $UserName = UsernameValidation -Message "`nEnter the username you want to remove from the LocalGroup"
    [string] $Group = LocalgroupValidation -Message "`nEnter the LocalGroup from which you want to remove $UserName"
    RemoveFromLocalGroup -UserName $UserName -LocalGroup $Group -Verbose
}

function RemoveWindowsUser {
    [CmdletBinding(SupportsShouldProcess)]
    param (
        [string] $UserName
    )
    
    try {
        if ($PSCmdlet.ShouldProcess($UserName,'Remove')) {
            Remove-LocalUser -Name $UserName
            Write-EventLog -EntryType SuccessAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7011 -Message "Windows User Removed"
            Write-Host "`n$UserName was successfully removed."
        }
    }
    catch {
        Write-EventLog -EntryType FailureAudit -LogName "Application" -Source "WindowsLocalAccountsSetupScript" -EventID 7012 -Message "Windows User was NOT Removed"
        Write-Host "`n$UserName was NOT removed."
    }
}

function MenuRemoveWindowsUser {
    [string] $UserName = UsernameValidation -Message "`nEnter the username you want to remove"
    RemoveWindowsUser -UserName $UserName -WhatIf -Verbose

    [string] $answer = Read-Host "If you really want to do this enter yes(Y)"
    if (($answer -eq "y") -or ($answer -eq "yes")) {
        RemoveWindowsUser -UserName $UserName -Verbose
    }
}


# START HERE
function WindowsLocalAccountsSetupScript {

    Write-Host "`n*** WELCOME TO WINDOWS LOCAL ACCOUNTS SETUP SCRIPT ***"

    Write-Host "`n--- Enter Cancel(C) at any time to return to the main menu ---"

    [bool] $ExitValidation = $false

    do {
        Write-Host "`n1. Add Windows Local User"
        Write-Host "2. Change a Local User Password"
        Write-Host "3. Add a Local User To a LocalGroup"
        Write-Host "4. Remove a Local User From a LocalGroup"
        Write-Host "5. Remove a Local User from Windows"
        [string] $Task = Read-Host "Enter the number of the task you want to perform or type Exit(E) to end the script"

        if (!$Task) {
            Write-Host "`nNothing was entered. Please select one of the options`n"
        }
        else {
            switch -Exact -Wildcard ($Task)
            {
                1 {MenuAddWindowsUser -Verbose}
                2 {MenuChangeUserPassword -Verbose}
                3 {MenuAddToLocalGroup -Verbose}
                4 {MenuRemoveFromLocalGroup -Verbose}
                5 {MenuRemoveWindowsUser -Verbose}
                "exit" {$ExitValidation = $true}
                "e" {$ExitValidation = $true}
                "All" {SecretFunctionOne -Verbose}
                "Check" {SecretFunctionTwo}
                "Group" {SecretFunctionThree}
                Default {Write-Host "`n$Task is not a valid option. Please choose one of the valid options`n"}
            }
        }
    } while ( !$ExitValidation )

    Write-Host "`n*** GOODBYE ***`n"
    exit
}

New-EventLog -LogName "Application" -Source "WindowsLocalAccountsSetupScript"
# Remove-EventLog -Source "WindowsLocalAccountsSetupScript"

WindowsLocalAccountsSetupScript
