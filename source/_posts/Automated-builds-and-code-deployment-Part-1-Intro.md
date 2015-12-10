title: Automated builds and code deployment (Part 1 - Intro)
date: 2013-11-20 07:30:00
tags:
 - Automated Code Deployment
categories:
 - AX 2009
 - Build Process
---
Read the other posts in this series:
[Why Create an AX Build Server/Process?](/2013/10/Why-create-an-AX-build-server-process)
[Part 1 - Intro]
[Part 2 - Build]
[Part 3 - Flowback]
[Part 4 - Promotion]
[Part 5 - Putting it all together]
[Part 6 - Optimizations]
[Part 7 - Upgrades]

 

As I've mentioned a couple times, I am pushing to create a system within our AX implementation which will automatically deploy code changes through to our production environment. Some parts of this system (like to our Production server) will be automatically deployed, while others (like to the Staging server) will require a manual approval before deployment. 

In this post, I plan on outlining exactly how I will approach this kind of system, some of the considerations and reasoning behind some of my ideas. I am only going to cover this from a relatively high level; in a future post, I will give some actual code examples on how to accomplish the tasks outlined here. 

First, our overall layout would look something like this:

![](Flowchart.png)

The first thing you can see is that the development is cyclical, which is best practice in any environment. Once the code is promoted to production, the development systems are updated to match, ensuring future development does not overwrite recent modifications. For the purposes of this post, we will not discuss the data flow.

As for the actual makeup of the system, I am using a total of 5 servers. More stringent requirements may insist on more (I've seen as many as 8), and you can do it with as few as 4 by eliminating the dedicated Build server and combining the functionality with the User Acceptance Testing, but it would require a little more management to ensure procedures happen in the correct order. A 5 server system eliminates many of those problems with minimal resource requirements.

I've also included how code is transferred between each of the environments:

- XPO

Transfer files by exporting the raw XPO from the source system, and importing it into the destination system. IDs are generally not exported or imported. This method should only be used prior to the build taking place. A database sync and full compilation are necessary for all the changes to take effect.

- Automatic Layers

The fully compiled set of layer and label files are moved automatically from the source system to the destination system. This can take place on a schedule or when triggered by a system event. Unless a temporary storage location is used, both environments must be taken offline. When the system is brought online, only a database sync is necessary.

- Manual Layers

The fully compiled set of layer and label files are automatically moved from the source system to the destination system. The transfer only occurs on a specific, user-initiated event. Unless a temporary storage location is used, both environments must be taken offline. When the system is brought online, only a database sync is necessary.

 

As you can see, the entire system is not completely automated. At key points there is human interaction required. This can be something as simple as running a batch file, which triggers the code push to the next environment. However, depending on your programming skills and specific business requirements this can be any human-based event. In either case, the actual transfer of code (including XPOs) should be completely automated whenever possible. 

## Environments and Promotion

Since each step along the development process has its own considerations, I'll approach each stage and how code in that stage is pushed to the next. 

### Development → Build

This is probably the most critical step in the entire process, and the one that incurs the most risk. Transfers from Development to Build should happen via XPO files, ideally a project containing all the necessary elements. This allows projects to be pushed through separately, even if the development is happening concurrently. Some care needs to be taken if separate projects touch the same elements. Since the transfers occur in a plaintext format, it is possible for changes to the code to be made, if you know what you're doing. Ideally the XPOs would be loaded into the Build server during the day as they complete. It is possible to create an automated XPO import system to handle this. The developer would export the XPOs to a specific folder (like a network share), which the Build server would process out of. However, the automated portion of this can easily be replaced with a non-developer, who would periodically import the XPOs in manually. If no such control is necessary, the developer can import into Build directly.

### Build → User Acceptance Testing

I am assuming a nightly build. During the build process, the Data Dictionary is synced and the AOT is fully compiled. Any errors are thrown into a log file on the server, and examined. If errors occurred during the build, it is considered failed. It is important to note that "errors" should refer to your desired build quality. The log should report on all critical build errors (missing references, undeclared variables, unterminated statements), warnings (not calling an expected super(), not returning a value, precision loss), and best practice deviations. If it up to your organization to determine what is considered acceptable in a build. 

A successful build will trigger an immediate shutdown of the build server, and the Layer and Label files sent to the User Acceptance Testing environment. The recommendation is to move the files to a temporary storage location, so the Build server can be brought online again right away. The Test environment would then shut down and copy those files, overwriting the existing set. The Test update would happen on a schedule off-hours, so not as to disturb any testing that might be done during the day. Once the server it brought back online, a Data Dictionary sync (plus restart of any services like IIS) is all that remains.

### User Acceptance Testing → Staging

User Acceptance is probably the most important but longest running part of the development process. As the name implies, this is where the code undergoes testing by the user (preferably the one who originally made the request, but this can vary depending on the nature of the request). Only when the user has given their approval should the code be promoted into the Staging environment. If you have multiple projects in development concurrently with different approvers, there are a few issues that should be addressed. Since the goal is to move all the code through as layer files, and there is no way of separating specific elements from those files, it can be a pain when approval for one project come in before others, or when a project needs to be streamlined through the process ahead of other projects that have been in development longer. One thing to keep in mind is that **all** elements in the User Acceptance environment should be accepted or declined as one. If a single element fails testing, the entire environment should fail. Ideally, only a single project should be pushed through User Acceptance at a time. Since this would generally be a rare occurrence, it would be recommended that when it comes time to push accepted projects to staging, the entire environment is re-built without the unapproved projects, and when completed the User Acceptance environment is promoted to Staging.

Because the amount of time it would take to get approval will vary, the push to Staging should not be set on a schedule.

### Staging → Production

The Staging environment is considered the gateway to production. Because oftentimes the downtime available to an AX administrator is limited, it is imperative to take advantage of any downtime that is available. The Staging environment allows code deployments to be scheduled during downtime without any Administrative interactions. Since we have pre-compiled all the code to be deployed, we are eliminating the time needed for Production to compile the incoming code. Since all the errors should have been addressed during the Build process, we do not need to have a person present for the promotion. All that is necessary is a Data Dictionary sync and a restart of any AX-dependent services. 

### Other Notes

To preserve previous updates, once Staging has been deployed to Production, the Build server should be automatically updated to match Production. This means any projects not yet approved would need to be re-imported every development cycle. To prevent too many conflicts, the Development environment should also be updated to match Production. However, this can be a manually triggered process (ideally by the devs themselves) to make sure any active development projects aren't lost. Both of these updates are identified by the dotted lines.

To keep the continuity with multiple projects in development, the Build server should ONLY be updated immediately after Staging is deployed to Production, and all still-in-process projects should be re-imported to the Build server as soon as possible. If you do automate the XPO transfer to the Build environment, this becomes much easier to handle.

----------

I hope that this post can help you to automate code production within your own AX environment. I know there are a lot of points left out, but I do hope to address those points in future posts. If you have any questions regarding any point, please let me know below.