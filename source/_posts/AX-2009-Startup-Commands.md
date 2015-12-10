title: AX 2009 Startup Commands
date: 2015-10-21 06:00:00
tags:
 - Development
categories:
 - AX 2009
---
Normally, I try to only post things here that are relatively unique and not widely published across the internet. However, this is a topic that I've had confusion with in the past and feel the need to address. AX startup commands are generally easily found elsewhere, and if you can't find them, it's not terribly difficult to find and understand the code within AX. But, there are some parts of it I have found confusing until a short time ago, and want to try to clarify in case anyone else has the same confusions. I'll also approach this from a perspective of adding a new command to the startup command list, `ExportAOT`. 

## How Startup Commands Are Used

The AX Client program, ax32.exe, allows for many flags, which change how the application is run. If any flags are not used, the program looks up the necessary values in the configuration file that was used to start the process or the values in the registry configuration (which is exposed via the AX Configuration Utility, axclicfg.exe). Startup commands in particular are defined by passing in a argument to the `-startupcmd=` flag on the command line, by defining a value to the startupcmd property in a configuration file, or by filling out a value in the "Command to run at application startup" field of the AX Configuration Utility (under the General tab).

Regardless of how the argument is passed into the client application, the format of the argument is `"command_parameter"`. It should be noted that only up to one parameter can be passed into the application by default (you can expand this if needed), and it is optional, depending on the command supplied. 

To clarify, a command should look like this (if executed from the command line):

```BASH
ax32.exe -startupcmd=AOTImport_"C:\AX\Import\File.xpo"
```

## Application Interpretation

The entire command (that is, `"command_parameter"`) is passed into the `\Classes\SysStartupCmd` class, specifically the `construct` static method. From there, the command is parsed to obtain the command and parameter, and a subclass extending the `SysStartupCmd` class is created and returned to the application. The subclass will ultimately define the behavior of the application as it starts up.

There are 4 primary methods that are called as the application is started up, and are called in the following order:
```a
applInit - Before the application is started
applRun - After the application has started running
infoInit - Before the infolog global controller is started
infoRun - After the infolog global controller is running
```

There is also a `canRun `method, which defines if the command should be allowed to execute. By default, commands can only be run if the user full access to the `SysDevelopment` security key or if the application is in setup/upgrade mode. This can be overridden in subclasses.

## Adding a Command

So, to add a command to the framework, we need two things: a subclass that extends `SysStartupCmd` which will be used to actually run the task, and an entry in the `SysStartupCmd::construct` method.

The second is easy - just insert a case branch in the main switch, which sets `sysStartupCmd` to a new instance of our subclass (passing in the parameters `s` and `parm`, like the others):

```axapta SysStartupCmd.construct
static SysStartupCmd construct(str startupCommand)
{
	.
	.
	.
    switch (s)
    {
		.
		.
		.
		case 'aotexport':
            sysStartupCmd = new SysStartupCmdExportAOT(s, parm);
            break;
    }
	.
	.
	.
}
```

All that's left is to define the class so it does what we want. Because we don't want it to run until after everything is loaded, we'll only need to override the infoRun method in our class:

```axapta SysStartupCmdExportAOT.infoRun
void infoRun()
{
    ExportAOT       export = new ExportAOT();
    str             exportPath;
    ;

    exportPath = parm ? parm : @"C:\Some\Default\Location";

    export.run(exportPath);
    this.shutDown();
}
```

We have our actual functionality extracted to another class (`ExportAOT`), so we'll just create a new instance and run it from our startup command. Because we want to have a default parameter in case one is not passed in, we also handle that as a simple ternary operation. Additionally, once we are done exporting, we want the client to automatically close, so we'll use the `shutDown` method provided in `SysStartupCmd` class to close the application. If we wanted to adjust the conditions that need to be met for the user to be able to run the command, we would also override the canRun method to check for the correct conditions. However, for our purposes the default requirements will work.

## List of Startup Commands

The following is a list of the startup commands that are present in the default AX 2009 system:

- SetBuildNo - Updates the version information text in the About Microsoft Dynamics AX dialog
- UpdateBuildNo
- Synchronize - Synchronizes the data dictionary
- CompileAll - Optional parameter "+"; + also updates cross-references
- Exit - Shuts down the application
- AotImport - Imports an XPO file into the AOT
- ApplUpgrade
- LoadLicense - Loads a license file into AX and shuts down
- CheckBestPractices - Compiles the application and exports the Best Practice warnings to a file. If an XPO file is provided, only check the Best Practice warnings in the XPO.
- AutoRun - Runs AX AutoRun tasks based on an XML file.
- RunTestProject - Creates, runs and closes a test project
- Batch - Starts the application to run Client batch tasks in a specific group
- ViewAlert - Opens a specific alert
- DrillDown - Opens the record that triggered an alert
- ViewAlertRule - Opens the rule that triggered an alert
- ImportResouces - Imports a resource into the AOT
- XmlReflection - Creates an XML file based on reflection of the AOT
- XmlDocumentation - Creates an XML documentation file
 