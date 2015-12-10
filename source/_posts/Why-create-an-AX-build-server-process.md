title: Why create an AX build server/process?
date: 2013-10-25 04:30:00
tags:
categories:
 - AX 2009
 - Build Process
 - Development
---
Read the other posts in this series:
Why Create an AX Build Server/Process?
Part 1 - Intro
Part 2 - Build
Part 3 - Flowback
Part 4 - Promotion
Part 5 - Putting it all together
Part 6 - Optimizations
Part 7 - Upgrades

This post is a precursor to another post I'm working on, but the information in it is too unique and involved to just add to that post. Instead, I'm putting this in its own post so it's easier to reference. 

In short, this is to answer one question I have, until recently, been struggling with: Why should an AX development and the deployment process consist of a build server?

Unlike most programming structures, the use of a build server or build process is not as intuitive for AX. However, after attending Summit this year, I'm beginning to understand that while it may not reach it's full potential when there is a small development team, it becomes incredibly helpful for larger teams. This doesn't mean you shouldn't use one in a small team, but with one or two developers it creates more overhead than it would for a large team.

A lot of the considerations revolve around standard development practices, and what the community has established as Best Practices. If you already have a development infrastructure in place (separate development, test, and pre-production environments), this can also be very easy to implement.

Originally, our primary way of transferring code between all environments was done via XPO files. There were some issues with this, mostly streaming from having multiple AOS instances, but we were able to overcome that by scheduling our code updates to work with an AOS restart schedule. Since we are publically traded, this also helped separate those that develop (me) from those who administer the production environment (my boss). 

Over the course of time, I began to learn some of the Best Practices used in an AX development environment - multiple environments dedicated to specific functions (Dev, UAT, pre-production), as well as effective code management using the binary .aod files.

However, everything really came together at Summit, when I learned that as Best Practice you should do a full system compilation on the destination system. That is, unless you move ALL the layer files from a system that has already been through a full compile. As long as no further code changes are made, you can use those files through the entire deployment process, meaning (assuming you follow Best Practice) you save time on every step of the process.