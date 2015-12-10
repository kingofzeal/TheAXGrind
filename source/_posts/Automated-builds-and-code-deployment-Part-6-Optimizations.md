title: Automated builds and code deployment (Part 6 - Optimizations)
date: 2014-12-29 07:30:00
tags:
 - TeamCity
 - Automated Code Deployment
 - Build
categories:
 - AX 2009
 - Build Process
---
Read the other posts in this series:
[Why Create an AX Build Server/Process?](/2013/10/Why-create-an-AX-build-server-process)
[Part 1 - Intro](/2013/11/Automated-builds-and-code-deployment-Part-1-Intro/)
[Part 2 - Build](/2013/12/Automated-builds-and-code-deployment-Part-2-Build/)
[Part 3 - Flowback](/2014/01/Automated-builds-and-code-deployment-Part-3-Flowback/)
[Part 4 - Promotion](/2014/10/Automated-builds-and-code-deployment-Part-4-Promotion/)
[Part 5 - Putting it all together](/2014/10/Automated-builds-and-code-deployment-Part-5-Putting-it-all-together/)
Part 6 - Optimizations
[Part 7 - Upgrades](/2015/12/Automated-builds-and-code-deployment-Part-7-Upgrades/)

Up until now I have only been discussing what it has taken to get a Lifecycle Management system in place for AX 2009. The system needs to be reliable and adaptable while providing a comprehensive set of automated tasks to allow changes to AX to be propagated correctly. And like most programming projects, it was approached as a "Let's get this in place so we have something as soon as we can". However, inefficiencies can and do add up, so now it's time to refactor our solution and make it more efficient.

For starters, we have replaced all of the scripts we use with PowerShell scripts for consistency. Since PowerShell is based on the .NET framework, we have the power and versatility of .NET. In some cases we are already using that power, but now we should really convert everything so there is consistency across the board.

In addition, we also noticed that many of scripts are redundant, with only minor changes from copy to copy. If something needs to change in the operations of one type of script, it would need to be changed in multiple places, increasing the chance of errors or inconsistencies. To fix that, we've generified majority of the scripts to so they are executed with parameters, which are defined by the specific build configurations. By extracting the differences to the TeamCity build configuration, we are able to turn each type of process (Promotion vs Flowback) into templates, allowing us to spin up a new process and only need to define the unique parameters for that process.

Instead of keeping the files segregated in their own process-specific folder, we've moved all of the generic scripts to the root of the version control repository:

![](NewDirectoryStructure.png)

We still have process-specific scripts, but those now only hold scripts that cannot be made generic, such as the SQL update scripts, which can't be made generic as easily as the process scripts. 

Here is an example of a script we have converted from a command line to PowerShell:

```powershell
[CmdletBinding()]
param(
    [Parameter(Mandatory=$True)]
        [string]$fileLocation)

Write-Debug "Removing cache files from $fileLocation"
$extensionList = @("alc", "ali", "aoi")

foreach ($extension in $extensionList)
{
    Remove-Item "$fileLocation\*.$extension"
    Write-Host "$extension files removed"
}
```

In this case, we are passing in a folder location (in our case, the network share for the specific environment), iterating over a list of extensions and removing all files with those extensions from that location. In addition to being able to run for any of our environments, it also allows us to easily remove additional file types by simply adding another entry to the list. If we wanted to remove a different list of extensions for each environments (or the ability to add additional extensions to the default on a per-environment bases), we could extract that to be a parameter as well. However, since our goal is to have each environment be as close to production as possible, we opted not to do that.

Here is another example where we can take advantage of advanced processing to allow the script to run for multiple types of environments:

```powershell
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
    [Parameter(Mandatory=$True)]
        [string]$serverName, 
        [string]$processName)

#-----Housekeeping-----
#serverName can be comma delimited. If it is, break it out to a list of server names
if ($serverName.Contains(',')) { $serverNameList = $serverName.Split(',') } else { $serverNameList = $serverName }

# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($processName)) { $processName = 'Dynamics AX Object Server 5.0$*' }
#----/Housekeeping-----

#For each server, start the process
foreach ($server in $serverNameList)
{
    Write-Debug "Starting $processName on $server"
    Write-Host "Starting $processName on $server"
    start-service -inputobject $(get-service -ComputerName "$server" -DisplayName "$processName") -WarningAction SilentlyContinue
    Write-Host "$processName on $server started"
}
```

This script accepts two parameters, the server name (which is mandatory) and the process name (which is optional). Additionally, it can accept multiple servers as a CSV string - we use this for our production environment, which is load balanced across three servers. The servers are started in the same order as they are passed in, and you only need to define the process name if it is different than the default "Dynamics AX Object Server 5.0$*" (for example, if you have two environments on the same server, so you only shut down one of those environments). We've also been able to include debug messages to verify what actions are occurring when changing the scripts, and confirm if you want the action to execute. These messages do not appear when executed by TeamCity.

On the TeamCity side, this script would be configured as follows:

![](TeamCityNewScriptSetup.png)

The `%AosServer%` in the script arguments section is a reference to a the configuration parameters of the build configuration:

![](TeamCityParmSetup.png)

Ultimately, these parameters drive the behavior of the entire process (which is why some parameters, like SqlServer, reference other parameters - because for this environment they are the same). 

&nbsp;
Finally, now that all the scripts are effectively applicable to all environments, it makes templating each of main processes easy, since all the scripts will take parameters. The parameters don't need to have a value within the template, they only need to be defined - the configurations provide the actual values. You can see from the screenshot above that majority of the parameters are inherited from a template. We have the option to reset any of them to the default values (in this case, blank, as defined on the template), but we cannot delete them.

Each of the configuration parameters is also configurable within TeamCity so a value must be provided. If no value is provided, the configuration will not execute. The nice side of configuring a template this way is spinning up a new process is as easy as clicking a button and filling in some values:

![](TeamCityTemplate.png)

&nbsp;
From there, the only things you need to define are the triggers and dependencies, if you need more than those that are defined via the template. 

Similar to the build scripts themselves, if there is a new process that needs to be added for each type of configuration (for example, a new step), we need only add it to the template, and it will automatically be added to all the configurations that inherit from that template.

 
&nbsp;
The goal of all this is to decrease the amount of maintenance that needs to be done when a change needs to be made. By standardizing the language of all the scripts, less overall knowledge is needed to manage them; if a script generates an error, we need only fix one version of the script instead of 5; if a process is missing a step, we need only change the template configuration instead of 3-4 build configurations. 

Here are samples for the scripts we have (not including the SQL files). Note that you may not need all these files:

```Powershell UpdateAxDatabase.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$sqlServerHost,
   [Parameter(Mandatory=$True)]
       [string]$sqlCloneScript,
       [string]$dbPermissionsScript,
       [string]$sqlDatabaseName)

#-----Housekeeping-----
# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($sqlDatabaseName)) { $sqlDatabaseName = "DynamicsAx1" }
#----/Housekeeping-----

Add-PSSnapin SqlServerCmdletSnapin100 -ErrorAction SilentlyContinue | Out-Null
Add-PSSnapin SqlServerProviderSnapin100 -ErrorAction SilentlyContinue | Out-Null

Write-Host "Dropping DB Connections"
$query = "ALTER DATABASE $sqlDatabaseName SET SINGLE_USER WITH ROLLBACK IMMEDIATE"
Write-Debug "QUERY: $query"
Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database "master" -Query $query -AbortOnError

Write-Host "Restoring database"
if ($sqlDatabaseName -ne "DynamicsAx1")
{
   $moveCmd = " MOVE N'DynamicsAx1' TO N'C:\Program Files\Microsoft SQL Server\MSSQL10_50.MSSQLSERVER\MSSQL\DATA\$sqlDatabaseName.mdf',  MOVE N'DynamicsAx1_log' TO N'C:\Program Files\Microsoft SQL Server\MSSQL10_50.MSSQLSERVER\MSSQL\DATA\${sqlDatabaseName}_log.LDF',"
}

$query = "RESTORE DATABASE [$sqlDatabaseName] FROM DISK = N'\\[network location]\DBBackup.bak' WITH FILE = 1,$moveCmd NOUNLOAD, REPLACE, STATS = 10"
Write-Debug "QUERY: $query"
Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database "master" -Query $query -AbortOnError -QueryTimeout 65535 -Verbose
Write-Host "Compressing database"

#Simple backup recovery
$query = "ALTER DATABASE [$sqlDatabaseName] SET RECOVERY SIMPLE"
Write-Debug "QUERY: $query"
Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database $sqlDatabaseName -Query $query -AbortOnError

#Shrink log file
$query = "DBCC SHRINKFILE (N'$DynamicsAx1_log' , 0, TRUNCATEONLY)"
Write-Debug "QUERY: $query"
Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database $sqlDatabaseName -Query $query -AbortOnError

#Execute database change script(s)
Write-Host "Applying Database Changes"
Write-Debug "Executing $sqlCloneScript"
Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database $sqlDatabaseName -InputFile $sqlCloneScript -AbortOnError -Verbose

if (![string]::IsNullOrEmpty($dbPermissionsScript))
{
   Write-Debug "Executing $dbPermissionsScript"
   Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database $sqlDatabaseName -InputFile $dbPermissionsScript -AbortOnError
}

#Enable multi-user in case restore failed
$query = "ALTER DATABASE $sqlDatabaseName SET Multi_user"
Write-Debug "QUERY: $query"
Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database $sqlDatabaseName -Query $query -AbortOnError
```
-------------------------------------------------------------------------------
``` Powershell CopyAosFiles.ps1 
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$destination)

Write-Debug "Copying files to $destination"
xcopy "AX Build Files\*.*" $destination /Y /z
```
-------------------------------------------------------------------------------
``` Powershell StartAxServers.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$serverName, 
       [string]$processName)

#-----Housekeeping-----
#serverName can be comma delimited. If it is, break it out to a list of server names
if ($serverName.Contains(',')) { $serverNameList = $serverName.Split(',') } else { $serverNameList = $serverName }

# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($processName)) { $processName = 'Dynamics AX Object Server 5.0$*' }
#----/Housekeeping-----

#For each server, start the process
foreach ($server in $serverNameList)
{
   Write-Debug "Starting $processName on $server"
   Write-Host "Starting $processName on $server"
   start-service -inputobject $(get-service -ComputerName "$server" -DisplayName "$processName") -WarningAction SilentlyContinue
   Write-Host "$processName on $server started"
}
```
-------------------------------------------------------------------------------
```Powershell BackupProdSql.ps1
$rootSourceLocation = "\\[Network location]"
$dbBackupFileName   = "DBBackup.bak"
$serverName         = "[Server Name]"
$dbBackupFile       = $rootSourceLocation + $dbBackupFileName
$query = "BACKUP DATABASE [DynamicsAx1] TO DISK = N'" + $dbBackupFile + "' WITH INIT, NOUNLOAD, NAME = N'DynamicsAx1 Clone Backup', NOSKIP, STATS = 10, NOFORMAT" 

sqlcmd -E -S $serverName -d master -Q $query | Out-Host
```
-------------------------------------------------------------------------------
``` Powershell CleanVcDirectory.ps1
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$fileLocation)

Write-Debug "Removing XPO files from $fileLocation"
Get-ChildItem $fileLocation -Recurse -Include "*.xpo" -Force | Remove-Item
```
-------------------------------------------------------------------------------
```Powershell CompileAx.ps1
ax32.exe -startupcmd=compileall_- | Out-Null
```
-------------------------------------------------------------------------------
```Powershell CopyVersionControlNotes.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)] [Int32] $dependencyBuildId,
   [Parameter(Mandatory=$True)] [string]$sourceSqlServer,
                                [string]$sourceSqlName,
   [Parameter(Mandatory=$True)] [string]$destinationSqlServer,
                                [string]$destinationSqlName)

#-----Housekeeping-----
# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($sourceSqlName))      { $sourceSqlName      = $sourceSqlServer      }
if ([string]::IsNullOrEmpty($destinationSqlName)) { $destinationSqlName = $destinationSqlServer }
#----/Housekeeping-----

Write-Debug "Transfering build $dependencyBuildId version control information from $sourceSqlServer ($sourceSqlName) to $destinationSqlServer ($destinationSqlName)"
Write-Host "##teamcity[message text='Loading build information for $dependencyBuildId' status='NORMAL']"

#Load XML assemblies
[Reflection.Assembly]::LoadWithPartialName("System.Xml.Linq") | Out-Null
[Reflection.Assembly]::LoadWithPartialName("System.Linq") | Out-Null
Add-PSSnapin SqlServerCmdletSnapin100 -ErrorAction SilentlyContinue | Out-Null
Add-PSSnapin SqlServerProviderSnapin100 -ErrorAction SilentlyContinue | Out-Null
#/assemblies

$buildInfo = [System.Xml.Linq.XDocument]::Load("http://[TeamCity URL]/guestAuth/app/rest/builds/id:$dependencyBuildId");
Write-Debug $buildInfo
$buildNum = $buildInfo.Root.Attribute("number").Value;
Write-Debug $buildNum
$buildDate = [DateTime]::ParseExact(($buildInfo.Root.Descendants("finishDate") | %{$_.Value}), "yyyyMMddTHHmmsszzz", [System.Globalization.CultureInfo]::InvariantCulture).ToUniversalTime().ToString("s");
write-debug $buildDate

if ($buildInfo.Root.Descendants("timestamp").Count -gt 0)
{
   $apprvDate = [DateTime]::ParseExact(($buildInfo.Root.Descendants("timestamp") | %{$_.Value}), "yyyyMMddTHHmmsszzz", [System.Globalization.CultureInfo]::InvariantCulture).ToUniversalTime().ToString("s");
   Write-Debug $apprvDate
}
else
{
   $apprvDate = Get-Date "1/1/1900 00:00:00"
}

#Update Build information in the environment
$query = "UPDATE SysEnvironment SET BUILDNO = $buildNum, BUILDDATE = '$buildDate', APPROVEDDATE = '$apprvDate'"
Invoke-Sqlcmd -ServerInstance $destinationSqlServer -Database "DynamicsAx1" -Query $query

#Pass along Version Control Items table
$query = "INSERT INTO [DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXITE2541] 
   SELECT DISTINCT src.*
       FROM [$sourceSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXITE2541] src
       LEFT JOIN [$destinationSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXITE2541] dest
           ON src.RECID = dest.RECID
       LEFT OUTER JOIN [$sourceSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXREV2543] rev
           ON rev.ITEMPATH = src.ITEMPATH
       WHERE dest.RECID IS NULL and rev.CREATEDDATETIME < '$buildDate'"
Invoke-Sqlcmd -ServerInstance $destinationSqlServer -Database "DynamicsAx1" -Query $query

#Pass along Version Control Lock table
$query = "INSERT INTO [DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXLOC2542] 
   SELECT src.*
       FROM [$sourceSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXLOC2542] src
       LEFT JOIN [$destinationSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXLOC2542] dest
           ON src.RECID = dest.RECID
       WHERE dest.RECID IS NULL and src.CREATEDDATETIME < '$buildDate'"
Invoke-Sqlcmd -ServerInstance $destinationSqlServer -Database "DynamicsAx1" -Query $query

#Pass along Version Control Revision table
$query = "INSERT INTO [DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXREV2543] 
   SELECT src.*
       FROM [$sourceSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXREV2543] src
       LEFT JOIN [$destinationSqlName].[DynamicsAx1].[dbo].[SYSVERSIONCONTROLMORPHXREV2543] dest
           ON src.RECID = dest.RECID
       WHERE dest.RECID IS NULL and src.CREATEDDATETIME < '$buildDate'"
Invoke-Sqlcmd -ServerInstance $destinationSqlServer -Database "DynamicsAx1" -Query $query

#Update RecID sequences for above tables
foreach ($i in (2541, 2542, 2543))
{
   $query = "UPDATE [DynamicsAx1].[dbo].[SYSTEMSEQUENCES]
       SET NEXTVAL = (SELECT NEXTVAL FROM [$sourceSqlName].[DynamicsAx1].[dbo].[SYSTEMSEQUENCES] src
           WHERE src.TABID = $i)
       WHERE TABID = $i"
   Invoke-Sqlcmd -ServerInstance $destinationSqlServer -Database "DynamicsAx1" -Query $query
}
```
-------------------------------------------------------------------------------
``` Powershell ExportAot.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$server,
   [Parameter(Mandatory=$True)]
       [string]$exportLocation,
       [string]$instanceName,
       [string]$instancePort)

#-----Housekeeping-----
#serverName can be comma delimited. If it is, only take the first in the list
if ($server.Contains(',')) { $server = $server.Split(',')[0] }

# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($instanceName))   { $instanceName   = "DynamicsAx1" }
if ([string]::IsNullOrEmpty($instancePort))   { $instancePort   = "2712"        }
#----/Housekeeping-----

Write-Debug "Exporting AOT From $instanceName@$server`:$instancePort to $exportLocation"

$args = @("-startupCmd=aotexport_$exportLocation", "-aos2=$instanceName@$server`:$instancePort")
ax32.exe $args | Out-Null
```
-------------------------------------------------------------------------------
``` Powershell MercurialCommit.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
    [Parameter(Mandatory=$True)]
        [int32] $buildNum,
    [Parameter(Mandatory=$True)]
        [string]$internalBuildId,
    [Parameter(Mandatory=$True)]
        [string]$sqlServerHost,
        [string]$sqlDatabaseName,
		[boolean]$pinned = $false)

#-----Housekeeping-----
# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($sqlDatabaseName)) { $sqlDatabaseName = "DynamicsAx1" }

$pinnedStr = if ($pinned -eq $true) {",pinned:true"} else {""}
#----/Housekeeping-----

#Load XML assemblies
[Reflection.Assembly]::LoadWithPartialName("System.Xml.Linq") | Out-Null
[Reflection.Assembly]::LoadWithPartialName("System.Linq") | Out-Null
Add-PSSnapin SqlServerCmdletSnapin100 -ErrorAction SilentlyContinue | Out-Null
Add-PSSnapin SqlServerProviderSnapin100 -ErrorAction SilentlyContinue | Out-Null
#/assemblies

$xmlFile = "http://[TeamCity URL]/guestAuth/app/rest/builds/buildType:(id:$internalBuildId),canceled:any,running:false$pinnedStr"
Write-Debug "Loading $xmlFile"
$buildInfo = [System.Xml.Linq.XDocument]::Load($xmlFile);
$buildDate = [DateTime]::ParseExact(($buildInfo.Root.Descendants("startDate") | %{$_.Value}), "yyyyMMddTHHmmsszzz", [System.Globalization.CultureInfo]::InvariantCulture).ToUniversalTime().ToString("s");

Write-Debug "Retrieving commits from $sqlServerHost.$sqlDatabaseName"
$commits = Invoke-Sqlcmd -ServerInstance $sqlServerHost -Database $sqlDatabaseName -Query "SELECT DISTINCT CAST(COMMENT_ as NVARCHAR(max)) FROM SYSVERSIONCONTROLMORPHXREV2543 WHERE CREATEDDATETIME > '$buildDate' ORDER BY CAST(COMMENT_ AS NVARCHAR(max)) ASC";
if (($commits | measure).Count -gt 0)
{
    $commits | ForEach-Object -Process { $commitMsgs = $commitMsgs + "`n" + $_.Column1 };
}
else
{
    # If there are no commit messages then the changes must be the 
    # automatic reversion due to unpinned builds 
    # NOTE: This will not show in a commit unless there are actual changes to be committed
    $commitMsgs = "Reverted unapproved or unpinned changes.";
} 

$message = "Build ${buildNum}: $commitMsgs"
Write-Debug "Adding to VC + Committing:`n$message"
hg addremove | Out-Host
hg commit -m "Build ${buildNum}: $commitMsgs" -u AXVControl | Out-Host
if (!$?) { return; }

Write-Debug "Pushing commit to remote repository"
hg push | Out-Host
```
-------------------------------------------------------------------------------
``` Powershell ParseCompilerResults.ps1
trap
{
	# On any thrown error, return with a non-zero exit code
    exit 1
}

if ($env:TEAMCITY_VERSION) {
        # When PowerShell is started through TeamCity's Command Runner, the standard
        # output will be wrapped at column 80 (a default). This has a negative impact
        # on service messages, as TeamCity quite naturally fails parsing a wrapped
        # message. The solution is to set a new, much wider output width. It will
        # only be set if TEAMCITY_VERSION exists, i.e., if started by TeamCity.
        $host.UI.RawUI.BufferSize = New-Object System.Management.Automation.Host.Size(8192,50)
}

[xml]$xml = (Get-Content "C:\Users\Public\Microsoft\Dynamics Ax\Log\AxCompileAll.xml")

$ns = @{ Table = "urn:www.microsoft.com/Formats/Table" }

$errorNodes = Select-Xml -XPath "/AxaptaCompilerOutput/Table:Record[Table:Field[@name='SysCompilerSeverity'] = 0]" -Xml $xml -Namespace $ns
$warningNodes = Select-Xml -XPath "/AxaptaCompilerOutput/Table:Record[Table:Field[@name='SysCompilerSeverity'] > 0 and Table:Field[@name='SysCompilerSeverity'] < 255]" -Xml $xml -Namespace $ns
$todoNodes = Select-Xml -XPath "/AxaptaCompilerOutput/Table:Record[Table:Field[@name='SysCompilerSeverity'] = 255]" -Xml $xml -Namespace $ns

$success = $true

if (($errorNodes | Measure-Object).Count -gt 0)
{
    foreach ($node in $errorNodes)
    {
        $success = $false
        $nodePath = ($node.Node.Field | ? { $_.name -eq "TreeNodePath" }).'#text'
        $message = ($node.Node.Field | ? { $_.name -eq "SysCompileErrorMessage" }).'#text'

        write-host "##teamcity[message text='${nodePath}: $message' status='ERROR']"
    }
}

foreach ($node in $warningNodes)
{
    $nodePath = ($node.Node.Field | ? { $_.name -eq "TreeNodePath" }).'#text'
    $message = ($node.Node.Field | ? { $_.name -eq "SysCompileErrorMessage" }).'#text'
    
    write-host "##teamcity[message text='${nodePath}: $message' status='WARNING']"
}

foreach ($node in $todoNodes)
{
    $nodePath = ($node.Node.Field | ? { $_.name -eq "TreeNodePath" }).'#text'
    $message = ($node.Node.Field | ? { $_.name -eq "SysCompileErrorMessage" }).'#text'
    
    write-host "${nodePath}: $message"    
}

if ($success -eq $false)
{
    throw $_
}
 ```
-------------------------------------------------------------------------------
``` Powershell RemoveAosCacheFiles.ps1
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$fileLocation)
Write-Debug "Removing cache files from $fileLocation"
$extensionList = @("alc", "ali", "aoi")

foreach ($extension in $extensionList)
{
   Remove-Item "$fileLocation\*.$extension"
   Write-Host "$extension files removed"
}
 ```
-------------------------------------------------------------------------------
``` Powershell StopAxServers.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
   [Parameter(Mandatory=$True)]
       [string]$serverName,
       [string]$processName)

#-----Housekeeping-----
#serverName can be comma delimited. If it is, break it out to a list of server names
if ($serverName.Contains(',')) { $serverNameList = $serverName.Split(',') } else { $serverNameList = $serverName }

# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($processName)) { $processName = 'Dynamics AX Object Server 5.0$*' }
#----/Housekeeping-----

#For each server, stop the process
foreach ($server in $serverNameList)
{
   Write-Debug "Stopping $processName on $server"
   Write-Host "Stopping $processName on $server"
   stop-service -inputobject $(get-service -ComputerName "$server" -DisplayName "$processName") -WarningAction SilentlyContinue
   Write-Host "$processName on $server stopped."
}
 ```
-------------------------------------------------------------------------------
``` Powershell SynchronizeDataDictionary.ps1
#Note: default values may not work, as empty strings may be passed. 
#  Such strings should be treated as if nothing was passed.
[CmdletBinding()]
param(
    [Parameter(Mandatory=$True)]
        [string]$server,
        [string]$instanceName,
        [string]$instancePort)

#-----Housekeeping-----
#serverName can be comma delimited. If it is, only take the first in the list
if ($server.Contains(',')) { $server = $server.Split(',')[0] }

# Set default values if nothing was passed in
if ([string]::IsNullOrEmpty($instanceName)) { $instanceName = "DynamicsAx1" }
if ([string]::IsNullOrEmpty($instancePort)) { $instancePort = "2712"        }
#----/Housekeeping-----

Write-Debug "Performing Data Dictionary Sync on $instanceName@$server`:$instancePort"
$args = @("-startupcmd=synchronize", "-aos2=$instanceName@$server`:$instancePort")
Ax32.exe $args | Out-Null
 ```

Working on AX Lifecycle Management has shown a lot as far as how the AX infrastructure has been set up. While I'm sure this system may only have limited use in AX 2012 or AX 7 (if any at all), having a formal process has allowed me as a developer to focus more of my time on actual development without having to worry about how it is going to be deployed into our production environments. Having an automated environment update process allows me to track how changes impact the environment more realistically, so code changes are less likely to have problems when they do finally reach production. The built-in audit trail is fantastic for showing to auditors, both internal and external, what happened and when. And while we haven't had the need to roll back an update it's nice to know the possibility still exists if it is needed.

Ultimately, the system we have built is a very simple one - easy to understand, easy to explain, easy to show what exactly is going on. But the impact it is making has been huge - mostly because of its simplicity. I hope this series inspires others to implement similar systems for their organizations.