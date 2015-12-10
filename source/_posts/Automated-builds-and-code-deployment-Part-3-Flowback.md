title: Automated builds and code deployment (Part 3 - Flowback)
date: 2014-01-08 13:45:00
tags:
 - Automated Code Deployment
 - Build
 - TeamCity
categories:
 - AX 2009
 - Build Process
---
Read the other posts in this series:
[Why Create an AX Build Server/Process?](/2013/10/Why-create-an-AX-build-server-process)
[Part 1 - Intro](/2013/11/Automated-builds-and-code-deployment-Part-1-Intro/)
[Part 2 - Build](/2013/12/Automated-builds-and-code-deployment-Part-2-Build/)
Part 3 - Flowback
[Part 4 - Promotion](/2014/10/Automated-builds-and-code-deployment-Part-4-Promotion/)
[Part 5 - Putting it all together](/2014/10/Automated-builds-and-code-deployment-Part-5-Putting-it-all-together/)
[Part 6 - Optimizations](/2014/12/Automated-builds-and-code-deployment-Part-6-Optimizations/)
[Part 7 - Upgrades](/2015/12/Automated-builds-and-code-deployment-Part-7-Upgrades/)

I'm going to skip what really should be my next post in the Automated Builds and Code Deployment series (which should be about how code is promoted from Dev -> Build -> UAT -> Staging -> Production), and instead write about the reverse process, which I will be referring to as flowback. 

Basically, since our environments need to be kept up to date with production, for both code and data, we need to have processes in place that allow us to update those development environments after certain conditions. In our case, we have 2 distinct development environments that need to be updated in this way: Build and Dev. All the other environments should be based on one of those two environments (or based on an environment that is based on those environments, etc).

Because we are still working through our own implementation of automated code deployment, we decided to implement the flowback before the promotion, since it would help with our existing manual promotion strategy. Once the promotion architecture is all set up, we can adjust the flowback processes as necessary, which will be fairly minor.

 

To manage our flowback processes, we are again using TeamCity. We are running TeamCity 8, and taking advantage of a new feature that was not present in version 7: Templates. Templates are a base build configuration, with steps, triggers, conditions, dependencies, parameters and requirements defined. When you create a new configuration based on a template, you are not able to delete anything that is defined in the template, but you can add additional attributes or change some of them. It is important if you want to use templates that not everything can be overridden. If in doubt, assume the template will override any settings on the configuration.

We've hooked our flowback template to be based on the same VCS repository that was mentioned in the previous post. This repository holds all the script files necessary to operate every aspect of the project: Build, Promotion and Flowback. We have it organized so each server has its own folder. If the server has both a promotion and a flowback process (Build in this case), it has a Promotion and Flowback folder as well. This structure will become helpful in a moment.

The flowback process itself is fairly simple, with only 4 steps: Shutdown the AX server process, update the server binaries, restore the SQL database, restart the AX server process.

![](TeamCityFlowbackProcess.png) 

Our production server still has a few operations that are run outside of the managed build process. Because we don't want to bring the production server offline every night to copy the binaries, we use a tool that takes a Shadow Copy of the files, and moves them on to a central network location. In addition, the production SQL database is backed up to the same location. Our managed flowback process will use these files to restore the environments.

 

Here is an example of what one of our steps look like:

![](TeamCityShutdownServer.png) 

You will notice the script name is hardcoded into the step (in this case '.\StopAxServers.ps1'). This is where the VCS folder structure comes in. In our template, we define a build parameter:

![](TeamCityWorkingDirectoryParm.png)

We do not specify a value on the template itself, we simply declare its existence. When a new configuration is created, it will prompt us for the value of this parameter, and the configuration will not run without it. The value is the relative path of the scripts in relation to the VCS root. So, for our Build Flowback, the value will be 'Build/Flowback'; for Build promotion to UAT it will be 'Build/Promotion'; for Dev flowback, it will simply be 'Dev'. Each flowback process has identically named files, with only the script commands changed as needed to reflect the different environments.

Since each flowback process should be identical except for a couple key variables (namely the destination server name), this makes things much easier to manage. If, in the future, we decide that all flowback processes should have another step, we simply add it to the template, ensure the new script file is in the correct directories and it will automatically show up the next time the configuration is run.

 

Each step in the template calls a single file:

Shutdown AX Server: StopAxServers.ps1
Update AX Source Files: UpdateAxFiles.ps1
Update AX Database: UpdateAxDatabase.bat
Start AX Server: StartAXServers.ps1


Each UpdateAxDatabase.bat file looks like this:

```dos UpdateAxDatabase.bat
@echo off
REM local resources
set sqlServerHost=[Destination SQL Server Name]
set sqlServerCloneScript=Server Clone.sql
REM /local resources

REM remote resources
set rootSourceLocation=[Network Database backup location]
set dbBackupFileName=DBBackup.bak
REM /remote resources

REM calculations
set dbBackupFile=%rootSourceLocation%%dbBackupFileName%
REM /calculations

echo Dropping DB Connections...
osql -E -S %sqlServerHost% -d master -Q "ALTER DATABASE DynamicsAx1 SET SINGLE_USER WITH ROLLBACK IMMEDIATE" -b
IF ERRORLEVEL 1 ECHO ##teamcity[message text='Failed to drop database connections' status='ERROR']

echo Restoring database...
set query=RESTORE DATABASE [DynamicsAx1] FROM DISK = N'%dbBackupFile%' WITH FILE = 1, NOUNLOAD, REPLACE, STATS = 10
osql -E -S %sqlServerHost% -d master -Q "%query%" -b
IF ERRORLEVEL 1 ECHO ##teamcity[message text='Failed to restore database' status='ERROR']

echo Compressing database
osql -E -S %sqlServerHost% -d DynamicsAx1 -Q "ALTER DATABASE [DynamicsAx1] SET RECOVERY SIMPLE" -b
IF ERRORLEVEL 1 ECHO ##teamcity[message text='Failed to set simple recovery mode' status='WARNING']

REM Simple backup recovery
osql -E -S %sqlServerHost% -d DynamicsAx1 -Q "DBCC SHRINKFILE (N'DynamicsAx1_log' , 0, TRUNCATEONLY)" -b
IF ERRORLEVEL 1 ECHO ##teamcity[message text='Failed to truncate database log' status='WARNING']

REM Shrink log file
echo Applying Database Changes...
osql -E -S %sqlServerHost% -d DynamicsAx1 -i "%sqlServerCloneScript%" -b
IF ERRORLEVEL 1 ECHO ##teamcity[message text='Failed to apply database changes' status='ERROR']

REM Execute database change script
osql -E -S %sqlServerHost% -d DynamicsAx1 -Q "ALTER DATABASE DynamicsAX1 SET Multi_user" -b
IF ERRORLEVEL 1 ECHO ##teamcity[message text='Failed to restore database connections' status='ERROR']

REM Enable multi-user in case restore failed
```

The script is set up to have the items that change (the hostname and file locations) at the top, and use them throughout the rest of the script. Feel free to adjust it as needed. Because of a bug with powershell scripts in TeamCity, it's difficult to capture any errors that occur. The osql command (with the -b flag) allows us to check the result of the SQL command and issue messages back to TeamCity. Depending on the command, it could fail the build (failing to restore the database will cause an error, but if we can't shrink the database it's not a big of a deal so we only issue a warning).

 The SQL Clone Script referenced is separate SQL file. The purpose is to fix URLs, user permissions, etc so the server operates correctly. This is a sample of our script (note: This is for AX 2009; this may not work in 2012 because of database schema changes):
 
 
```SQL SQL Clone Script
/* Old Server Information */
Declare @P_AOS  varchar(15)
Declare @P_SQL  varchar(15)
Declare @P_SSRS varchar(15)

Set @P_AOS =  '{Production AOS Server}'
Set @P_SQL =  '{Production SQL Server}'
Set @P_SSRS = '{Production SSRS Server}'
 
/* New Server Information */
Declare @AOS  varchar(15)
Declare @SQL  varchar(15)
Declare @SSRS varchar(15)
 
Set @AOS =  '{Development AOS Server}'
Set @SQL =  '{Development SQL Server}'
Set @SSRS = '{Development SSRS Server}'
 

--Remove old server connections ----------------------------------------------------
TRUNCATE TABLE [SYSSERVERCONFIG] 
TRUNCATE TABLE [SYSSERVERSESSIONS]

--Update SSRS Setting---------------------------------------------------------------
Update SRSSERVERS Set [SERVERID] = REPLACE(upper([SERVERID]), upper(@P_SSRS), upper(@SSRS)) 
Where upper([SERVERID]) like '%' + upper(@P_SSRS) + '%'

Update SRSSERVERS Set [SERVERURL] = REPLACE(upper([SERVERURL]), upper(@P_SSRS), upper(@SSRS)) 
Where upper([SERVERURL]) like '%' + upper(@P_SSRS) + '%'

Update SRSSERVERS Set [REPORTMANAGERURL] = REPLACE(upper([REPORTMANAGERURL]), upper(@P_SSRS), upper(@SSRS))
Where upper([REPORTMANAGERURL]) like '%' + upper(@P_SSRS) + '%'

--Update SQL Settings---------------------------------------------------------------
Update [SYSCLIENTSESSIONS] Set CLIENTCOMPUTER = REPLACE(upper(CLIENTCOMPUTER), upper(@P_SQL), upper(@SQL)) 
Where upper(CLIENTCOMPUTER) like '%' + upper(@P_SQL) + '%' 

Update [SYSLASTVALUE] Set [DESIGNNAME] = REPLACE(upper([DESIGNNAME]), upper(@P_SQL), upper(@SQL)) 
Where upper([DESIGNNAME]) like '%' + upper(@P_SQL) + '%' 

Update SYSUSERLOG Set [COMPUTERNAME] = REPLACE(upper([COMPUTERNAME]), upper(@P_SQL), upper(@SQL)) 
Where upper([COMPUTERNAME]) like '%' + upper(@P_SQL) + '%' 
------------------------------------------------------------------------------------
UPDATE SysSQMSettings Set GlobalGUID = '{00000000-0000-0000-0000-000000000000}'
------------------------------------------------------------------------------------
UPDATE [SYSBCPROXYUSERACCOUNT] SET [SID] = 'S-x-x-xx-xxxxxxxxx-xxxxxxxxx-xxxxxxxxxx-xxxxx', [NETWORKALIAS] = '{Development Business Connector Proxy Account}'
where [networkalias] = '{Production Business Connector Proxy Account}'

------------------------------------------------------------------------------------
-------------------USER AND PLUGIN CHANGES------------------------------------------
------------------------------------------------------------------------------------

--Disable all users
UPDATE USERINFO Set [ENABLE] = 0 WHERE ID != 'admin'

--Re-enable all dev users
UPDATE USERINFO Set [ENABLE] = 1 
WHERE ID IN ({User list})
UPDATE USERINFO Set [SID] = 'S-x-x-xx-xxxxxxxxx-xxxxxxxxx-xxxxxxxxxx-xxxxx' WHERE ID = 'admi3'
.
.
.
 
--Set compiler to level 2
UPDATE USERINFO SET [COMPILERWARNINGLEVEL] = 2 WHERE [COMPILERWARNINGLEVEL] > 2
------------------------------------------------------------------------------------

--Assign new permissions as needed
INSERT INTO USERGROUPLIST ([USERID],[GROUPID],[MODIFIEDDATETIME],[DEL_MODIFIEDTIME],[MODIFIEDBY],[CREATEDDATETIME],[DEL_CREATEDTIME],[CREATEDBY],[RECVERSION],[RECID])
VALUES 
     ('{user}','Admin','2012-01-01 12:00:00.000',0,'-AOS-','2012-01-01 12:00:00.000',0,'-AOS-',1,5637148952)
.
.
.
 
------------------------------------------------------------------------------------
--UPDATE SYSVERSIONCONTROLPARAMETERS Set [VCSEnabled] = 1 WHERE KEY_ = 0
------------------------------------------------------------------------------------
```

In addition to modifying user settings and permissions, we also manage things like 3rd party plugins that should be pointed to different environments, and obfuscate certain information that should only be readily available in production. We also have a modification in our system that allows us to change the background of the windows on a company-by-company basis to easily identify which environment/company you are working in. When this script is run, the color is set to Red across all companies (Dev is Purple, Staging is Green, UAT is orange).

When the server is finally started back up, everything is repointed to the development settings, and it now operates independently of production as it should.