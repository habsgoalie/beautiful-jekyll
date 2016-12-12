---
layout: post
title: Checking DSC Nodes for compliance with configurations 
subtitle: A quick and easy script you can use to monitor your DSC nodes in Azure Automation.
comments: true
---

If you are using Azure Automation DSC Pull Service, you probably want to have some service checks in place. Part of the power of DSC is keeping everything in a **Desired State**. If there's configuration drift, you'd like to know about it! You also want to know if nodes are failing to check in at all. The Azure Portal provides a nice portal to quickly view your DSC estate, but it doesn't natively provide alerting.

![alt text](/img/dsc-portal.png "All's well here.")

There are several ways to do this, and if you use [OMS](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwiUiMOPr-3QAhUT9WMKHV97AfsQFgg6MAI&url=https%3A%2F%2Fwww.microsoft.com%2Fen-us%2Fcloud-platform%2Foperations-management-suite&usg=AFQjCNEUKa6AOBdCRBzolRq3HDp8dCK47A&sig2=GVBmaLqx4HCRIyYme8tb-Q) or [System Center Ops Manager](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=12&cad=rja&uact=8&ved=0ahUKEwjO-PGqr-3QAhVS2WMKHf2RBe0QFghYMAs&url=https%3A%2F%2Ftechnet.microsoft.com%2Fen-us%2Flibrary%2Fhh205987(v%3Dsc.12).aspx&usg=AFQjCNEQ5YxHjJ4rutLXB3-3Sqa9orCiIQ&sig2=kg0auC-TYIgX4xRTnLBuwg) those tools have monitoring for Azure Automation DSC built in. This is great for some, but at my company we are standardized on non-Microsoft monitoring tools, as I am sure some of you are. Don't despair though! You can also build some simple checks using PowerShell.

The script I am using makes use of the [Azure RM PowerShell Module](https://msdn.microsoft.com/en-us/library/mt125356.aspx?f=255&MSPPError=-2147217396), and can be run using an Azure Automation [Runbook](https://docs.microsoft.com/en-us/azure/automation/automation-starting-a-runbook) or an automation tool of your choice.

## The Script, explained

For this workflow, I want a simple email alert to be sent to me and my team when a DSC consistency check runs and there is an issue. I want to know if the consistency check fails for some reason, or if the node fails to report status (unresponsive). 

>**Note:** DSC Configuration modes are important to mention here. There are 3 modes, *Apply Only*, *Apply and Monitor*, and *Apply and AutoCorrect*. The latter 2 will force consistency checks at intervals. Obviously, if you choose *Apply and AutoCorrect*, the need to monitor and alert may be less, since the DSC engine will try to autocorrect configuration drift. However, If you use *Apply and Monitor*, configuration drift will be noted on the pull service but not corrected, so that's where you really need some alerting.

There are several cmdlets in the AzureRM module that can query status of DSC nodes.

```powershell
Get-AzureRmAutomationDscNodeReport
```
This one seems perfect. The only parameters it requires are ```ResourceGroupName``` and ```AutomationAccountName```. If you run it with just those, it will return all reports on all nodes. Thats going to be a **lot** of data in some long running environments! You can also run it with the boolean value ```-Latest``` . However, in order to run this against a specific node, you need to no the NodeID, which is a unique id assigned by the pull service. It's not very human readable.

I need to find a way to expose the human-friendly server name of the node. Let's try this:

```powershell
Get-AzureRmAutomationDscNode 
```
This one get's the node as an object, and contains a property called ```Name``` which is what I need. Here's what the output looks like:

```

ResourceGroupName     : dscprod
AutomationAccountName : dscprod
Name                  : MYSERVER01
RegistrationTime      : 5/6/2016 3:47:32 PM -04:00
LastSeen              : 12/11/2016 11:59:31 PM -05:00
IpAddress             : 192.168.1.5
Id                    : 44422252-c500-11e5-80cf-ef3309844c1
NodeConfigurationName : nodeconfig.clusternode01
Status                : Failed
```

Look at that - the last property is ```Status```. Just what I need! I think this command is perfect to build the script around.

First we will construct a section to check for the **AzureRM** module we need and install it if needed, then log in.

```powershell
$Name = "AzureRM"
$Module = Get-Module -Listavailable |  Where-object { $_.Name.Contains($Name) }
If(!($Module)){
	Install-Module $Name -Force
    Import-Module -Name $Name
}
Else{
    Import-Module -Name $Name
}

Login-AzureRmAccount -Credential $Credential
```

Next we will build an array of node objects from the ```Get-AzureRmAutomationDscNode``` cmdlet.

```powershell
$Nodes = Get-AzureRmAutomationDscNode -AutomationAccountName $AutomationAccountName -ResourceGroupName $ResourceGroupName
```

Lastly, we create some functions to check for the conditions we want to alert on, ```Failed``` and ```Unresponsive```.

```powershell
Function Check-Unresponsive{
    $Unresponsive = ForEach($Node in $Nodes){
        If($Node.Status -eq "unresponsive"){
            $Node
        }
    }
    If($Unresponsive){
        $Body = $Unresponsive | Out-String
        Send-Mailmessage -to $Recipient -Subject "DSC Check - Unresponsive Node(s)" -from $Sender -Body $Body -SmtpServer $MailServer
    }
}

Function Check-Failed{
    $Failed = ForEach($Node in $Nodes){
        If($Node.Status -eq "failed"){
            $Node
        }
    }
    If($Failed){
        $Body = $Failed | Out-String
        Send-Mailmessage -to $Recipient -Subject "DSC Check - Failed Node(s)" -from $Sender -Body $Body -SmtpServer $MailServer
    }
}
```

And that's the script! If you want the full script you can download it from my [Github repository](https://github.com/habsgoalie/DscComplianceChecks). 
Thanks for reading, and as always, let me know if you found this helpful or not in the comments!