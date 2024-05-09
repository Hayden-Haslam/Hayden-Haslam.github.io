# May 3, 2024

## Current Project: Making a script to automate wim extraction and setup

**Problem to solve:** I need to recreate Windows images every quarter for
security and PCI compliance reasons. My current solution to this was to mount my
existing `install.wim` and boot it into audit mode, make whatever changes are
needed, and reapply any sysprep stuff, and recapture. This became very tedious
very fast with most of the steps being done manually.

**Goal:** I want to create a nice script to do most of the steps for me. I will
out line it first with my rough order and where I think the commands need to go
for this process. Then I will slowly go through and actually uncomment the
commands and make the thing real!

**Update #1:** As I've been working through this, I've made a lot of progress on
the logic of the script. I have also been trying to incorporate use of
`Write-Verbose` to leverage some verbose output for troubleshoot purposes, but I
am still a little confused on how it exactly works. So for now, disregard the
excessive amount for `Write-Verbose` lines.

**Update #2:** Ive made a lot of changes here. In the future, I will try to do a
better job of showing my old way and new way side by side instead of replacing
everything at once. But I started practicing the use of Advanced Functions and
creating my own cmdlets. I have converted this script into the `New-MaverikImage`
cmdlet. I have made two parameters for the two locations that will be needed on
the computer. My goal is to have a network location that is accessible to
everyone running my script. I set that location as default parameter value, but
they can change it if they wish. Same goes for the local location on their
computer where the actual image building/manipulation will happen.

Since the `Mount-WindowsImage` cmdlet requires Administrator privileges, I have
added `#requires -RunAsAdministrator` to my function as well. I still have much
more to do with this script but I feel like I am making very good progress.

```powershell
#requires -RunAsAdministrator

function New-MaverikImage {
    [CmdletBinding()]
    param(
        [Parameter(HelpMessage = "Enter the location on you local computer where you want to make the working directory.")]
        [string] $LocalDirectory = 'C:\zzz',
        [Parameter(HelpMessage = "Enter the location that you want to pull the working files from.")]
        [string] $NetworkLocation = 'C:\VM\workingdirctoryshare'
        #[Parameter(Mandatory)]
        #[pscredential] $Credentials
    )

    # Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ]"
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Starting Script."
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Setting Working Directory to $LocalDirectory."
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Testing if $LocalDirectory exists."
    if (!(Test-Path -Path $LocalDirectory -PathType Any)) {
        Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] $LocalDirectory does not exist, creating now."
        #New-Item -Path 'C:\' -Name 'workingdirectory' -ItemType Directory
        New-Item -ItemType Directory -Path $LocalDirectory
        Write-Warning -Message "The workingdirectory folder was not present. It has now been created."
    }
    
    Copy-Item -Path $NetworkLocation\* -Destination $LocalDirectory -Recurse

    $ISOLocation = (Get-Item -Path $LocalDirectory'\iso\*.iso').FullName
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Setting ISO location: $ISOLocation."
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Testing if ISO exists."
    if (!(Test-Path -Path $ISOLocation -PathType Leaf)) {
        Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] ISO does not exist, display Warning and exit."
        Write-Warning -Message "Iso file is missing. Copy SW_DVD9_WIN_ENT_LTSC_2021_64BIT_English_MLF_X22-84414.ISO into C:\workingdirectory\iso and restart script."
        Return
    }

    $DriverFolder = $LocalDirectory+'\Drivers'
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Setting Driver folder location: $DriverFolder."
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Testing if Driver folder exists."
    if (!(Test-Path -Path $DriverFolder -PathType Any)) {
        Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] $DriverFolder does not exits, creating now."
        New-Item -Path $LocalDirectory'\' -Name 'Drivers' -ItemType Directory
        Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Display Warning and exit."
        Write-Warning -Message "It looks like the driver folder is empty, add drivers and restart script"
        Return
    }

    # Mount iso file
    $LocationInfo = (Mount-DiskImage -ImagePath $ISOLocation -PassThru | Get-Volume)
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Mounting ISO for wim extraction."
    $DriveLetter = ($LocationInfo).DriveLetter+':\'
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] ISO mounted at Driver Letter: $DriveLetter"
    $DevicePath = ($LocationInfo | Get-DiskImage).DevicePath
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] ISO mounted at Device Location: $DevicePath"
    $WimName = 'Win10Enterprise.wim'
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Setting extracted wim name: $WimName"

    # Extract LTSC Edition from install.wim in sources directory to working directory
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Extracting Windows 10 LTSC version from install.wim to $LocalDirectory"
    Dism /Export-Image /SourceImageFile:$DriveLetter\sources\install.wim /SourceIndex:1 /DestinationImageFile:$LocalDirectory\iso\$WimName

    # Unmount iso file
    Write-Verbose -Message "[ $((Get-Date).TimeOfDay.ToString()) ] Dismount ISO file from $DevicePath"
    Dismount-DiskImage -DevicePath $DevicePath

    # Rename the wim index to our own thing
        # Using dism run below
        # Imagex /info "<PATH TO IMAGE>" <Index Number> "<NEW IMAGE NAME>" "<NEW IMAGE DESCRIPTION>"
        # Impossible unless ran from the Deployment tools utility

    # Mount wim file
    New-Item -Path $LocalDirectory -Name 'MountFolder' -ItemType Directory
    $MountFolder = $LocalDirectory+'\MountFolder'
    $WimLocation = $LocalDirectory+'\iso\'+$WimName
    Mount-WindowsImage -ImagePath $WimLocation -Index 1 -Path $MountFolder

    # Inject device drivers
    dism /Image:$MountFolder /Add-Driver /Driver:$DriverFolder /Recurse

    <# Windows Updates from WSUS
        # Set Server
        $WSUSServer = 'PC01.lab.test' # eventually on Legend
        # Set OS type
        $OSName = 'Windows 10 LTSB' # Might need to fix

        # Install Update tools
        Install-WindowsFeature UpdateServices-API

        # Set folder for downloading updates
        $UpdateFolder = New-Item -Path $LocalDirectory -Name 'Updates' -ItemType Directory

        # Connect to WSUS server
        $WSUSConnection = Get-WsusServer -Name $WSUSServer -PortNumber 8530
        $AllUpdates = $WSUSConnection.GetUpdates()

        $OSUpdates = $AllUpdates | Where-Object {$OSName -in $_.ProductTitles -and $_.IsApproved -eq $true -and $_.IsSuperseded -eq $false}

        foreach($Update in $OSUpdates) {
            $CabURIs = $Update.GetInstallableItems() | Select-Object -ExpandProperty Files | Where-Object {$_.Type -eq 'SelfContained'} | Select-Object -ExpandProperty FileUri | Select-Object -ExpandProperty AbsoluteUri
            foreach($CabURI in $CabURIs) {
                # $CabFile = $CabURI | Split-Path -Leaf
                $CabPath = $CabURI.Replace('/','\').Replace('http:','').Replace(':','').Replace(':8530','\C$').Replace('Content','WSUS\WsusContent')
                Copy-Item -Path $CabPath -Destination $UpdateFolder
            }
        }

        $UpdateFiles = Get-ChildItem $UpdateFolder | Select-Object -ExpandProperty FullName

        foreach ($AvailableUpdate in $UpdateFiles) {
            Add-WindowsPackage -PackagePath $AvailableUpdate -Path $MountFolder
        }
    #>

    # Make Audit Mode answer file
        # To configure Windows to boot to audit mode, add the Microsoft-Windows-Deployment | Reseal | Mode = audit
        Copy-Item -Path $LocalDirectory'\iso\unattend.xml' -Destination $MountFolder'\Windows\System32\Sysprep'
        Copy-Item -Path $LocalDirectory'\iso\MaverikOOBE' -Destination $MountFolder'\Windows\System32\Sysprep' -Recurse
        # Run script on first time start up
            # Install-Module -Name PSWindowsUpdate
            # Get-WindowsUpdate -AcceptAll

    # Copy sysprep files into sysprep folder in image
        # Copy-Item

    # Unmount wim file
    Dismount-WindowsImage -path $MountFolder -save

    # Rename wim file
        # Rename-File -Path <path to install.wim> -NewName <New Name of wim>

    # Hyper-V automated setup
        # Build Virtual Machine
        # ? Mount VHD 
        # $vhdfile = $LocalDirectory+'\iso\temp.vhdx'
        # New-VHD -Path $vhdfile -Dynamic -SizeBytes 30GB | Mount-VHD -Passthru | Initialize-Disk -Passthru -Confirm:$false
        # $disknumber = (Get-DiskImage -ImagePath $vhdfile | Get-Disk).Number
        # ? Run Diskpart commands to format the drive correctly
        # ? Expand-WindowsImage and <cmd.exe /c "bcdboot W:\Windows /s S: /f ALL"> to apply wim to vhd drive
        # ? Unmount vhd drive
        # ? Assign vhd to Virtual Machine
        # ? Start Virtual Machine

    # Boot Virtual Machine
        # Audit Mode boots automatically and runs Startup script

    # In startup Script
        # Check for windows updates - Install Windows update module from PSGallery
        # Reconfigure sysprep folder for actual install - New answer file, etc.
        # ? Other Stuff?
        # ? Disk clean up, remove old windows installs, etc.
        # ? Figure out a way to get through multiple reboot automatically for updates
        # Shutdown Image with generalize and sysprep ready to go

    # Capture VHD to wim
        # might need to do some fancy winpe stuff

    # Test booting the image to verify successful configuration
        # Verify sysprep processes correctly
        # Verify start up scripts process correctly

    # Clean up
        # ? Maybe remove all Hyper-V stuff
        # If using WDS, copy new image to WDS
}
```
