---
title:  "CaptainMon! - Process Spwaning tool"
date:   2020-06-15 19:14:18
categories: [Windows_WMI]
tags: [Process_Spwaning, WMI]
---

In this post we will be looking at my recent develpment on Windows Process Spwaning using CaptainMon.

Prior knowledge on the following topics are required:
1. Windows Management Instumentation.
2. WQLEventQuery.
3. ManagementScope.

```ps1
if(fail to have the above required knowledge)
{
    Write-host "Don't worry, you can learn that in this post:";
}
```

Windows Management Instrumentation:

Windows Management Instrumentation (WMI) is the infrastructure for management data and operations on Windows-based operating systems. You can use WMI from client applications and scripts. It provides an infrastructure that makes it easy to both discover and perform management tasks. In addition, you can add to the set of possible management tasks by creating your own WMI providers.

WQLEventQuery:

Represents a WMI event query in WQL format. Following Constructor related to WQLEventQuery is used in CaptainMon.

WqlEventQuery(String, String, TimeSpan): This initializes a new instance of the WqlEventQuery class with the specified event class name, condition, and grouping interval.

Following implementation is made in CaptainMon for defining an Event Query object in powershell:

```ps1
$query = New-Object System.Management.WQLEventQuery("__InstanceCreationEvent",$new_process_check_interval,"TargetInstance ISA 'Win32_Process'" );
```
Implementation of the same in c# is shown below:

```c#
WqlEventQuery query =
            new WqlEventQuery("__InstanceCreationEvent",
            new TimeSpan(0,0,1),
            "TargetInstance isa \"Win32_Process\"");
```
Note: WMI-query in-c# does not work on non-english machine.

ManagementScope:

Represents a scope (namespace) for management operations. Used to make a connection to a remote computer, following Constructor related to ManagementScope is used in CaptainMon.

ManagementScope(ManagementPath): This initializes a new instance of the ManagementScope class representing the specified scope path.

Implemention of ManagementScope() object in CaptainMon:

```ps1
$scope = New-Object System.Management.ManagementScope("\DESKTOPJ3BH2\.\root\cimV2");
```
Implemention of the same in c#:

```C#
ManagementScope scope =
            new ManagementScope(
            "\\\\FullComputerName\\root\\cimv2");
        scope.Connect();
```
Summing up the Object implementation together by including the Culture in Powershell:

```ps1
$scope = New-Object System.Management.ManagementScope("\DESKTOPJ3BH2\.\root\cimV2");
$query = New-Object System.Management.WQLEventQuery("__InstanceCreationEvent",$new_process_check_interval,"TargetInstance ISA 'Win32_Process'" );
$watcher = New-Object System.Management.ManagementEventWatcher($scope,$query);
```

Now Spwaning the new processes repetedly by incrementing the counter and using a Sychronous call for the watcher:

```ps1
$scope = New-Object System.Management.ManagementScope("\DESKTOPJ3BH2\.\root\cimV2");
$query = New-Object System.Management.WQLEventQuery("__InstanceCreationEvent",$new_process_check_interval,"TargetInstance ISA 'Win32_Process'" );
$watcher = New-Object System.Management.ManagementEventWatcher($scope,$query);

$capatainmon=1;
do
{
    captainmon += 1;
}
while($true)
```
Now spwaning the newly arrived event:

```ps1
$NEvent = $watcher.WaitForNextEvent();
$Ti = $newlyArrivedEvent.TargetInstance;
$processName=[string]$Ti.Name;
```
Writing Parent_Process to the host:

```ps1
$parent_process=''; 
	try 
	{
		$proc=(Get-Process -id $Ti.ParentProcessID -ea stop); 
		$parent_process=$proc.ProcessName;
	} 
	catch 
	{
		$parent_process='Unknown';
	}
```
Suspending the Process:

```ps1
function Sp($processID) {
	if(($pProc -ne [IntPtr]::Zero){
		Write-Host "Trying to suspend process: $processID"

		$result = SuspendProcess($pProc)
		if($result -ne 0) {
			Write-Error "Failed to suspend. SuspendProcess returned: $result"
			return $False
		}
		CloseHandle($pProc) | out-null;
	} else {
		Write-Error "Unable to open process. Not elevated? Process doesn't exist anymore?"
		return $False
	}
	return $True
}
if (-not ($ignoredProcesses -match $processName))
	{
		if(Suspend-Process -processID $Ti.ProcessId)
		{
			Write-Host "Process is suspended";
		}
	}
```
Resuming the Process:

```ps1
function Rp($processID) {
	if(($pProc -ne [IntPtr]::Zero){
		Write-Host "Trying to resume process: $processID"
		Write-Host ""
		$result = ResumeProcess($pProc)
		if($result -ne 0) {
			Write-Error "Failed to resume. ResumeProcess returned: $result"
		}
		[Threader]::CloseHandle($pProc) | out-null
	} else {
		Write-Error "Unable to open process. Process doesn't exist anymore?"
	}
}
```

Finally writing the output to the host:
```ps1
Write-host "";
	$Ti.processName, $Ti.ProcessId, $Ti.ParentProcessID, $parent_process | Out-File - FilePath .\Output.json
```

Thanks For Reading