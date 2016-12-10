---
layout: post
title: Bootstrapping Jenkins Windows slave nodes with DSC 
subtitle: A tasty recipe for serving up Windows Jenkins nodes using Chocolatey, Jenkins Swarm Client, and DSC!
comments: true
---

About a year ago I was looking for a way to centralize and manage scheduled jobs across our environment. I was turned on to the idea of using [Jenkins](https://jenkins.io/) after I stumbled across this [awesome blog](https://hodgkins.io/automating-with-jenkins-and-powershell-on-windows-part-1) courtesy of Matthew Hodgkins. Jenkins is an ubiquitous CI tool used in the world of DevOps for software build automation, but due to its modular plugin based design, its flexible enough to use as an automation engine for just about anything you want. 

![alt text](/img/jenkinsLogo1.png "Nice Moustache!")

Rather than convince you of the merits of using Jenkins for any type of job scheduling or automation, particularly for PowerShell workflows, I encourage you to stop now and read Matt's blogs ([part 1](https://hodgkins.io/automating-with-jenkins-and-powershell-on-windows-part-1), [part 2](https://hodgkins.io/automating-with-jenkins-and-powershell-on-windows-part-2)) on the topic. Once you are completely sold on the capabilities of Jenkins, come back here and continue reading this post to learn how to automate the thorniest issue with Jenkins - deploying Windows slaves.

In order to use Jenkins to run PowerShell worflows, you need Windows slave systems which the Jenkins master can execute code on. 

The [official installation process](https://wiki.jenkins-ci.org/display/JENKINS/Step+by+step+guide+to+set+up+master+and+slave+machines) for a Java slave agent service on a Windows computer requires a lot of manual steps and coaxing of things like browser settings, java security, and in some cases manual file changes. It is not a very quick process, and with many manual steps, it's prone to error and inconsistency. Ever since deploying Jenkins to run PowerShell jobs and workflows, I have been looking for an answer to this java-based Windows rats nest. The answer has come from several directions. 

In this post I will walk you through each ingredient in this recipe, then fit them all together into a single solution using Desired State Configuration. I go over each piece in a fair amount of detail, so if you want to skip all the reading you can stop here and check out a completed sample config that I have [uploaded to Github](https://github.com/habsgoalie/DscJenkins). You can fork or use that as a starting point. Otherwise, read on!

## The first (and Main) ingredient - Chocolatey!

[Chocolatey](https://chocolatey.org/) is **the original**  package manager for Windows. Chocolatey brings the power of one liner terminal package installation, akin to apt-get or yum, to Windows environments. It's so powerful that Microsoft co-opted the idea and incorporated their own version of a package manager into Windows Server 2016. Despite this, Chocolatey is still the original game in town and arguably the better solution. 

Chocolatey will be used in this recipe to install the necessary prerquisites for our Windows slaves, namely Java and Git. We will use DSC to do it, but if you just wanted to install the packages quickly via command line, you could with the following:

```powershell
choco install jr8 git
```
...And wait a few minutes and your prerequisite packages will be installed.

![alt text](/img/choco_output.png "It's like an easy button!")

To do these steps with DSC we will use the community maintained DSC Resource [cChoco](https://github.com/PowerShellOrg/cChoco). But more on that later. For now, we have used Chocolatey to install some dependencies, but what about the Jenkins agent itself? Unfortunately this is a bit more complex than a chocolatey package. We need a few more ingredients.

## Second ingredient - Jenkins Swarm

The [standard process](https://wiki.jenkins-ci.org/display/JENKINS/Step+by+step+guide+to+set+up+master+and+slave+machines) for adding a Jenkins slave is to create a node on the master, then associate the actual machine with the node you created using a custom JNLP launcher url on the target machine. If only there was a way for slave machines to locate the master and self register. As it happens, there is! 

I mentioned at the top of this post how Jenkins' plugin-based modularity was what made it so compelling. So, of course there is a plug-in to address this need. Its called [Jenkins Swarm](https://wiki.jenkins-ci.org/display/JENKINS/Swarm+Plugin).

The Swarm Plugin consists of 2 parts - a listener, installed on your master as a plugin package, and a special version of the Jenkins agent to be installed on the Windows server. You could stop here- but the swarm client is not the same as the standard JNLP package, there is no "install-as-a-service" option. That means your WIndows slave will not be reboot resilient, or recoverable without manual intervention if something interrupts the java thread on your slave. So close!

Alas, we need another ingredient.

## Third Ingredient - Winsw.exe - A wrapper to make your slave a service

[Winsw.exe](https://github.com/kohsuke/winsw) is the glue to binds this package together. I got the idea on how to put it all together by following this [blog post](http://tech.akom.net/archives/100-Jenkins-Swarm-Slaves-on-Windows-using-Puppet.html), from a guy trying to achieve the same result, only using Puppet instead of my CM tool of choice, DSC. Following Mr. Puppet admin's lead, here is what I did:

* Download the latest [Swarm jar file](https://repo.jenkins-ci.org/releases/org/jenkins-ci/plugins/swarm-client/)
* Download the latest [Winsw binary](https://github.com/kohsuke/winsw/releases/download/PR-133/winsw.exe)
* Create a config file called jenkins.xml (sample xml below)
* Rename Swarm jar to `jenkins.jar`
* Rename Winsw exe to `jenkins.exe`
* Put all these things together in a folder, named jenkins, on a network share with public read access.

Here's the sample xml config -

```xml

<service>
  <id>jenkins</id>
  <name>Jenkins</name>
  <description>This service runs Jenkins continuous integration system.</description>
  <env name="JENKINS_HOME" value="%BASE%"/>
  <executable>java</executable>
  <arguments>-Xrs -Xmx256m -jar "%BASE%\jenkins.jar"  -master 'https://yourjenkinsmasterhere.local/' -executors 1 -fsroot 'C:\Jenkins' -name %COMPUTERNAME% -labels 'SWARM' -disableSslVerification -disableClientsUniqueId</arguments>
  <logmode>rotate</logmode>
</service>

```

We are going to use DSC file resource to copy these files from the network to our target machine later. But to test, you could copy these files, with a well-formed xml file, and run the command

```
.\jenkins.exe install
```
and your service would be registered and installed. The xml file would establish standard paramaters and environment variables, as well as tell the agent where to find it's master. We will do this with DSC (notice the theme here?) I suppose it's time to start actually writing a DSC configuration!

## Putting it all together - Desired State Configuration

OK. We've gotten this far. We have all the ingredients. Now, let's put them together in a nice and tidy DSC config. We will start by importing some resources that we will need. [DSC Resources](https://msdn.microsoft.com/en-us/powershell/dsc/resources) are special PowerShell modules that let you add to the built-in configuration options of DSC. We will be using a mix of community and Microsoft authored resources for this configuarion. Community resources are typically denoted with a small c name, while Microsoft reosurces are either lacking a preceding letter or denoted by a small x - for eXperimental. The resources we will import are - 

* [cChoco](https://www.powershellgallery.com/packages/cChoco/2.3.0.0) - for installing Chocolatey packages
* [xCredSSP](https://www.powershellgallery.com/packages/xCredSSP/1.1.0.0) - for configuring the Windows slave to deal with PowerShell [Double-Hop](http://www.colinbowern.com/posts/overcoming-double-hop-authentication-powershell-remoting) issues
* [cNtfsAccessControl](https://www.powershellgallery.com/packages/cNtfsAccessControl/1.3.0) - for assigning proper permissions to the jenkins directory we will be copying

You will need to add these resources to your targets. You can do it manually on a machine with wmf5 by running 

```powershell
Install-Module cChoco
Install-Module xCredSSP
Install-Module cNtfsAccessControl
```

If you are using a Pull server, you will need to download and stage the modules on your pull server (see [this tutorial](https://www.penflip.com/powershellorg/the-dsc-book/blob/master/deploying-resources-via-pull-servers.txt)). If you are using [Azure Automation DSC](https://docs.microsoft.com/en-us/azure/automation/automation-dsc-overview), like I am, you can just import the modules directly from the PowerShell Gallery using the Azure portal.

![alt text](/img/azure-portal-screenshot.png "Get your fresh hot gallery modules!")

Here's what to add in your configuration script - 

```powershell

    Import-DscResource -ModuleName cChoco
    Import-DscResource -ModuleName xCredSSP
    Import-DscResource -ModuleName cNtfsAccessControl
```

The next part we will cover is using the cChoco resource. cChoco can install Chocolatey itself, and then be used to install additional Chocolatey packages. You just need to know the name, and optionally the version you want to install. My configuration example is how you would do this using the default community Chocolatey repository. I'm using packages that have passed public moderation by the Chocolatey community, but if you need extra security around your package sources, you can specify an internal Chocolatey repository as the source for your packages as well. I will write a future blog post on how to do this, but for now I will keep with the public feed to make things simple. Here's what the code looks like: 

```powershell

cChocoInstaller installChoco
        {
            InstallDir = "C:\choco"
        }

        cChocoPackageInstaller installGit 
        {            
            Name = "git" 
            DependsOn = "[cChocoInstaller]installChoco"
        }

        cChocoPackageInstaller installJava 
        {            
            Name = "jre8" 
            DependsOn = "[cChocoInstaller]installChoco"
        }
```

Let's break down what's happening here. In the first block, I am using cChoco's `cChocoInstaller` function to install Chocolatey in the directory c:\choco.
Next, I use the `cChocoPackageInstaller` function to install both git and java (known by its package name jre8). 

Notice that I only need to specify the package name, and the `DependsOn` block. This block allows me to specify that the DSC configuration engine should wait until Chocolatey is installed before trying to use it to install the other packages. This is important - The DSC configuration engine will decide on it's own the best order to run your configuration in unless you specify dependencies.

The next thing we need to do is copy the Jenkins folder with the renames swarm and winsw files in it on to our computer. To do this we will use the built-in DSC [File](https://msdn.microsoft.com/en-us/powershell/dsc/fileresource) resource. This will allow us to copy the file from the network location we left it to a local directory of our choosing, in this case C:\Jenkins.

```powershell

file installJenkins
        {
            Ensure = "Present"
            Type = "Directory"
            Recurse = $true
            SourcePath = "\\myfileserver\Packages\Jenkins"
            DestinationPath = "C:\Jenkins"
        }

```

Here we use the `Ensure = "Present"` key/value pair to make sure the directory gets copied into place if it is missing on the target machine. The rest of the options should be self-explanatory.

Next we have some extra steps that I find to be necessary to make this work smoothly. The first is ensuring the .Net framework 3.5 features are enabled on the box. This seems to be a requirement of the Winsw.exe install process, and doesnt work without these features enabled. Luckily, we can use the built-in [WindowsFeature](https://msdn.microsoft.com/en-us/powershell/dsc/windowsfeatureresource) resource to enable them.

```powershell

windowsFeature dotNet35Features
        {
            Ensure = "Present"
            Name = "NET-Framework-Core"
            IncludeAllSubFeature = $True
        }
```

This is simple like the cChoco configuration - just specify the feature by it's PowerShell name and use `Ensure = "Present"` to make sure its added.

We also need to make sure the permissions on the c:\Jenkins directory are agreeable for Jenkins operations. We do this using the cNtfsAccessControl resource.

```powershell

cNtfsPermissionEntry jenkinsPerm
        {
            Ensure = 'Present'
            Path = "C:\Jenkins"
            Principal = 'BUILTIN\Users'
            AccessControlInformation = @(
                cNtfsAccessControlInformation
                {
                    AccessControlType = 'Allow'
                    FileSystemRights = 'Modify'
                    Inheritance = 'ThisFolderSubfoldersAndFiles'
                    NoPropagateInherit = $false
                }
            )
            DependsOn = '[file]installJenkins'
        }

```

This lengthy configuration block simply makes certain the BUILTIN\Users Group can modify the contents of this folder. This will be required the first time your Jenkins master executes a job against this node and needs to build a workspace folder under c:\Jenkins with the LOCALSYSTEM account.

Now that our files are in place, we are finally ready to configure service installation and start. Earlier in the article I explained that Winsw.exe has an `install` switch that automatically installs the service for you. In order to call the executable and its switch, we need to use the built-in [Script](https://msdn.microsoft.com/en-us/powershell/dsc/scriptresource) resource. The script resource allows you to execute any block of PowerShell code within a configuration, and is a handi "multi-tool" to have in DSC if you can't find a particular resource to do what you need. Here's the code: 

```powershell

script configureJenkinsService
        {
            SetScript = {
                & 'C:\Jenkins\jenkins.exe' install
                }
            TestScript = {
                $Jenkins = Get-Service jenkins -ErrorAction SilentlyContinue
                if($Jenkins.Name -like "jenkins" -and $Jenkins.Status -like "Running"){
                    Write-Verbose "Jenkins Service is installed and running"
                    Return $True
                }
                else{
                    Write-Verbose "Jenkins Slave not running"
                    Return $False
                }
            }
            GetScript = {
                #Do Nothing
            }
            DependsOn = "[file]installJenkins"
        }

```

Ooh boy, let's dive into this one. The first block, `SetScript`, contains the command we want to run. But, DSC is a configuration engine, and it wants to assess the state of the machine its configuring before firing off any old script block. So, we have the `TestScript` block with a simple test to see if the `SetScript` action is needed or not. In this case - it's checking if there's any such service already running by the name of "Jenkins". If no such service is present, the script block will run, otherwise, the configuration engine will move on to other things. 

This logic is actually the common pattern of DSC resources themselves. If you look at the source of a DSC resource, they all contain common elements like `Get-TargetResource`, `Test-TargetResource`, and `Set-TargetResource`. The logic behind these constructs is the same as you are seeing here.

The last thing we need to do with our Jenkins service is make sure we leave it in a running state We use the built-in [Service](https://msdn.microsoft.com/en-us/powershell/dsc/serviceresource) resource.

```powershell

 Service jenkinsService
        {
            Ensure = "Present"
            Name = "jenkins"
            StartupType = "Automatic"
            State = "Running"
            DependsOn = "[script]configureJenkinsService"
        }
```

This one is also hopefully self-evident as well. We want our service present, running, and we use `DependsOn` key once again to make sure the service isn't started before the configuration element that installs it has run.

There are a couple more loose ends we need to tie up to make this DSC configuration robust. These are somewhat specific to my use case, you may not need these unless you are going to run PowerShell scripts that operate under any account context besides the SYSTEM account.

For us, many of our scripts need to run with a domain account, typically a service account purpose built for the job. This is possible using Jenkins, but you have to invoke commands as if you were using remoting in order for it to work, and you will quickly get into "Double Hop" trouble with this technique. To solve double-hop issues we enable [CredSSP](https://msdn.microsoft.com/en-us/powershell/reference/5.1/microsoft.wsman.management/enable-wsmancredssp) authentication mode using the xCredSSP resource.

```powershell

xCredSSP Client
        {
            Ensure = "Present"
            Role = "Client"
            DelegateComputers = "*.domain.local"
        }
```
Here we ensure CredSSP is enabled, and that the target machine is configured to both send and receive delegated credentials for our domain.

Lastly, we add our service account used for running PowerShell Jobs as an admin (or whatever built-in group is necessary) on the target machine. We use the built-in [Group](https://msdn.microsoft.com/en-us/powershell/dsc/groupresource) resource for this.

```powershell

Group addToAdministrators
        {
                GroupName = "Administrators"
                Ensure = "Present"
                MembersToInclude = "domain\svcuser"
                Credential = $Cred
        }
```

This one requires credentials, which you can (and should) store in a `pscredential` object when you compile this configuration into a mof file. This is **not recommended** unless you have set up ssl certificates for your Pull server or are using Azure Automation DSC as a Pull Service, which automatically manages certificates and encryption. Without this your mof files are produced in clear text, and include any passwords passed in, even if you use a secure method during compilation.

And Finally, we have a complete DSC configuration - when applied it will install everything needed, including the agent service itself, and self register to your specified master server, ready for jobs. Enjoy, and if you found this walkthrough helpful, let me know in the comments!


