# PowerShell + CSV

## The Why

I was recently assigned a new role/project where I will be administering a set of servers that run a complex program for my company. One of my first tasks as part of my training was to take some data from a csv file, containing 4 pieces of data per object, and generate an xml configuration file to be imported. The rest of the details aren't really important here, but my take away from this was how awesome it was to use a csv file as a source of data for a PowerShell script. Day 1 into training with this Administrator I will be replacing, and I have already learned something new. I have already thought of a way I can apply this to my current role. That is what I will be covering in this post.

## Goal

As mentioned above, I will be creating a PowerShell script that will run on a newly created machine in my environment that will reach out to an available network share, import the csv containing various configuration data, and do something with it to set up the device.

In the past, I found myself hard-coding things in my scripts like application installer names, local and network file destination paths, etc. I thought I could build a master cvs file, containing all the information relevant to the setup process and put in a place that is unlikely to change. Then, I only need to worry about managing the master spreadsheet, and make less changes to multiple scripts.

The part I am going to focus on for now is the program installation phase. This part of the csv will contain the name of the application that will be installed, the network location to pull it from, and the local destination on the computer to copy the file to.

## Example CSV

In my environment, I have computers that serve multiple functions and need to be set up accordingly, even though they use the same windows image and start up scripts. I will label these as type_1 and type_2 here.

| device_type | application_name | network_location | local_directory |
| :-: | :-: | :-: | :-: |
| type_1 | Application 1 | App1\* | C:\temp\software\App1 |
| type_1 | Application 2 | App2\* | C:\temp\software\App2 |
| type_2 | Application 3 | App3\* | C:\temp\software\App3 |
| type_2 | Application 4 | App4\* | C:\temp\software\App4 |

## The Script

I am going to start by setting a variable for my network share. This will be the only network directory that will be hard-coded into my script because it is unlikely to change.

```powershell
$networkshare = '\\networkshare\software\'
```

Next, I will create a `New-PSDrive` as my connection point to the network share so I only need to authenticate once. Note: the credentials are being gathered from another part of the script.

```powershell
New-PSDrive -Name 'Share' -PSProvider Filesystem -Root $networkshare -Credential $Credentials
```

Import the csv file.

```powershell
$csvpath = 'C:\temp\csvpath\test.csv'
$csv = Import-Csv -Path $csvpath
```

Time for the file transfers. I am using `Start-BitsTransfer` here for the progress bar since I plan on incorporating this into a larger script which includes a GUI.

```powershell
$csv | Where-Object device_type -eq 'type_1' | ForEach-Object {
    $source = "Share:\"+($_).network_location
    if (!(Test-Path $_.local_directory)) {New-Item -Path $_.local_directory -ItemType Directory}
    Start-BitsTransfer -Source $source -Destination $_.local_directory
}
```

Remove PSDrive when done.

```powershell
Remove-PSDrive -Name 'Share'
```
