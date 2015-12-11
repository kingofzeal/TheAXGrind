title: Automated builds and code deployment (Part 7 - Upgrades)
date: 2015-12-04 09:30:00
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
[Part 6 - Optimizations](/2014/12/Automated-builds-and-code-deployment-Part-6-Optimizations/)
Part 7 - Upgrades

Upgrades are a natural part of life when dealing with software, and AX is no exception. If you're still running AX 2009, the upgrades may have tapered off in favor of AX 2012 or even AX 7, but there are still patches released on a reasonable basis. If you have an automated code deployment system in place, however, these upgrades may be more complicated.

The way Microsoft distributes their updates is via an executable which deploys changes not only to the application files (normally the SYP layer), but in some cases to the actual server or application executable itself. Since our automated code deployment deals with the application files, the SYP changes will be propagated as normal, but we still need to manage the application executables. There is some added difficulty in that both the application executables and application files should be deployed around the same time to avoid either being out of sync.

We recently decided to upgrade our system from 2009 RU6 to RU8. The patch itself wasn't to really fix issues we had, but to ensure we were compatible with some 3rd party software we were looking at. Prior to the deployment, we deployed RU8 to a test environment and performed functionality testing so we knew what we were getting into.

When it came time to deploy, we ensure that it would be the only thing that was applied to our Build system (we put a hold on development for a couple weeks). Since the patch would still have to go through our normal gauntlet of environments before reaching production, this allowed us to roll back in case we found a stopping issue.

Prior to the automated deployment to each environment, we applied the RU8 patch to the environment. The automation then would overwrite the changed layer files with those from Build (which should be the same), and everything would be in sync post-deployment. Because we wanted the timing to be reasonably close, our team had to work closely to ensure everything was on track. To ensure nobody used the environment before the deployment had a chance to run, we unchecked the option to restart the application server when the upgrade was complete. The automated process would then turn it back on as part of its normal operation.

Finally, when it came time to upgrading the production cluster, we came in after-hours before the normal scheduled update and did the same as we did to the pre-production environments: leaving them offline after the patch, and letting the automated deployment turn them back on. We did have to schedule downtime because of the extended time needed to apply the patch, but our users were well-informed and we didn't have any issues.

In short, the automated build process can still help deploy any patches you need to apply to AX, but care must be taken to ensure the application executables are upgraded at the same time as the application files, or else you risk errors in the environment.