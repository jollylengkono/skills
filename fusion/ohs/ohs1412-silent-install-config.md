---
title: "Knowledge Article"
source: "https://support.oracle.com/support/?anchorId=&kmContentId=3070109&page=sptemplate&sptemplate=km-article"
author:
published:
created: 2026-05-30
description:
tags:
  - "clippings"
---
How to Install and Configure Standalone Oracle HTTP Server 14.1.2 in a Silent Mode(without GUI)

KB676483

Last Updated

Nov 14, 2025

Service

Oracle HTTP Server

---

Be the first to rate this article

Copyright © 2025, Oracle. All righs reserved. Oracle Confidential.

## Applies To

All Users  
Oracle HTTP Server - Version 14.1.2.0.0 and later

## Summary

The document explains the steps required to install and configure Oracle HTTP Server(OHS) in a silent mode without using GUI.

## Solution

### Check supported hardware and software configuration

Check the below 14.1.2 certification matrix and system requirements guide to confirm all requirements are met.  
[https://www.oracle.com/middleware/technologies/fusion-certification.html](https://www.oracle.com/middleware/technologies/fusion-certification.html "FMW 14.1.2 Certification guide")  
[https://www.oracle.com/middleware/technologies/internet-application-server/fusion-requirements-and-specifications.html](https://docs.oracle.com/en/middleware/fusion-middleware/14.1.2/sysrs/index.html "System Requirements and Specifications")

### Download 14.1.2 OHS software:

OHS 14.1.2 can be downloaded from:  
[https://www.oracle.com/uk/middleware/technologies/webtier-downloads.html](https://www.oracle.com/uk/middleware/technologies/webtier-downloads.html "Oracle Web Tier Downloads")

### Download and Install certified JDK:

If the requirement is to use custom JDK, download and install certified JDK.  
FMW 14.1.2 is supported on Java 17 and 21.

If custom JDK is not required, the below JDK which comes with the install can be used.  
<Oracle\_Home>>/oracle\_common/jdk

### Installing Oracle HTTP Server in a Silent Mode:

Run the below command to install OHS in a silent mode.

(UNIX)./fmw\_14.1.2.0.0\_ohs\_linux64.bin -silent -responseFile /<path\_to>/ohs.rsp

(WINDOWS) setup\_fmw\_14.1.2.0.0\_ohs\_win64.exe -silent -responseFile <path\_to>\\ohs.rsp

here "ohs.rsp" is the response file that contains install details:

The sample response file looks as below:

\[ENGINE\]

#DO NOT CHANGE THIS.  
Response File Version=1.0.0.0.0

\[GENERIC\]

#Set this to true if you wish to skip software updates  
DECLINE\_AUTO\_UPDATES=true

#My Oracle Support User Name  
MOS\_USERNAME=

#My Oracle Support Password  
MOS\_PASSWORD=<SECURE VALUE>

#If the Software updates are already downloaded and available on your local system, then specify the path to the directory where these patches are available and set SPECIFY\_DOWNLOAD\_LOCATION to true  
AUTO\_UPDATES\_LOCATION=

#Proxy Server Name to connect to My Oracle Support  
SOFTWARE\_UPDATES\_PROXY\_SERVER=

#Proxy Server Port  
SOFTWARE\_UPDATES\_PROXY\_PORT=

#Proxy Server Username  
SOFTWARE\_UPDATES\_PROXY\_USER=

#Proxy Server Password  
SOFTWARE\_UPDATES\_PROXY\_PASSWORD=<SECURE VALUE>

#The oracle home location. This can be an existing Oracle Home or a new Oracle Home  
**ORACLE\_HOME** =/<path\_to>/Standalone\_ohs\_home

#The federated oracle home locations. This should be an existing Oracle Home. Multiple values can be provided as comma seperated values  
FEDERATED\_ORACLE\_HOMES=

#Set this variable value to the Installation Type selected as either Standalone HTTP Server (Managed independently of WebLogic server) OR Collocated HTTP Server (Managed through WebLogic server)  
INSTALL\_TYPE=Standalone HTTP Server (Managed independently of WebLogic server)

#The jdk home location.  
**JDK\_HOME** =/<path\_to>/JDK\_HOME

**Note:** If the requirement is to use default JDK that comes with the install then Jdk home looks as below in the response file.  
JDK\_HOME=/<path\_to>/Standalone\_ohs\_home/oracle\_common/jdk

Below are the last few lines from the command prompt that confirms successful installation of standalone OHS.

Validations are enabled for this session.  
Verifying data  
Copying Files  
Percent Complete: 10  
Percent Complete: 20  
Percent Complete: 30  
Percent Complete: 40  
Percent Complete: 50  
Percent Complete: 60  
Percent Complete: 70  
Percent Complete: 80  
Percent Complete: 90  
Percent Complete: 100

The installation of Oracle HTTP Server 14.1.2.0.0 completed successfully.  
Logs successfully copied to /<path\_to>/oraInventory/logs

.

### Steps to Create Standalone OHS Domain Using WLST

1\. Navigate to ORACLE\_HOME>/oracle\_common/common/bin and run the command:

(UNIX)./wlst.sh  
(WINDOWS) wlst.cmd

2\. Once connected offline, execute below commands:

selectTemplate("Oracle HTTP Server (Standalone)","14.1.2.0.0")  
loadTemplates()  
cd('/')  
setOption('JavaHome','/<path\_to>/JDK\_HOME')  
create('<Domain\_Name>', 'SecurityConfiguration')  
cd('SecurityConfiguration/<Domain\_Name>')  
set('NodeManagerUsername', '<nodemanager\_username>')  
set('NodeManagerPasswordEncrypted', '<nodemanager\_password>')  
setOption('NodeManagerType', 'PerDomainNodeManager')  
cd('/')  
cd('/OHS/ohs1') ===========>By default template would contain ohs1 system component and one can set listen address and ports as shown in next steps  
cmo.setAdminHost('<Admin\_Host>') ======>This should be **local IP address 127.0.0.1**  
cmo.setAdminPort('<Admin\_Port>')  
cmo.setListenAddress('<Listen\_Address>')  
cmo.setListenPort('<OHS\_non\_ssl\_port>')  
cmo.setSSLListenPort('<OHS\_ssl\_port>')  
writeDomain('/<path\_to>/domain\_home')  
closeTemplate()  
exit()

### Start the servers and processes

After a successful install, start all services by following below steps:

1\. Start Node Manager

(UNIX) DOMAIN\_HOME/bin/startNodeManager.sh  
(Windows) DOMAIN\_HOME\\bin\\startNodeManager.cmd

2\. Start System Components using

(UNIX) DOMAIN\_HOME/bin/startComponent.sh component\_name  
(Windows) DOMAIN\_HOME\\bin\\startComponent.cmd component\_name

## References

MOS document id: 3070109.1

## Product Versions

product: Oracle HTTP Server - min\_version: 14.1.2.0.0 - max\_version: none; Information in this article applies to GENERIC (All Platforms)

## Internal Only

Created from <SR 3-39517846851>

## Authoring Instructions

This is a Phase 2e Migration article and should be edited in MOS.  
Do not make changes in MOSFS, and do not publish this article.  
MOS\_DOCUMENT\_ID: 3070109.1

## Article Feedback

Rate this

Knowledge Article <iframe aria-label="Click here to launch Oracle Guided Learning Help Panel" src="https://guidedlearning.oracle.com/player/launcher/v2/app/nyHZF90UQ4qEe+x1x72_wQ/lang/--/?draft=false&amp;min=true&amp;mobile=false&amp;allowKeyboardShortcuts=undefined&amp;debug=30">Launcher</iframe> <iframe allow="clipboard-write" aria-label="Oracle Guided Learning Help Panel" src="https://guidedlearning.oracle.com/player/contentWidget/v2/app/nyHZF90UQ4qEe+x1x72_wQ/lang/--/?draft=false&amp;min=true&amp;mobile=false&amp;allowKeyboardShortcuts=undefined&amp;debug=30&amp;helpHotKey=Alt%2BCtrl%2BH&amp;guideHotKey=Alt%2BCtrl%2BG&amp;displayGHroupHotKey=Alt%2BCtrl%2BM&amp;languageSelectionHotKey=Alt%2BCtrl%2BL">Content list</iframe>