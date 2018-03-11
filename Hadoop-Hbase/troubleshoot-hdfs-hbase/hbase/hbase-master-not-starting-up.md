---
title: Why is the HBase Master (HMaster) failing to startup? | Microsoft Docs
description: Use the HBase FAQ for answers to common questions on HBase on Azure HDInsight platform.
keywords: Azure HDInsight, HBase, FAQ, troubleshooting guide, common problems, hbase master (HMaster), not starting up
services: Azure HDInsight
documentationcenter: na
author: duoxu
manager: ''
editor: ''

ms.assetid: 82921B4B-DA11-44A5-885F-6A2FEBEC5FF6
ms.service: multiple
ms.workload: na
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 03/30/2017
ms.author: duoxu
---

### Why is the HBase Master (HMaster) failing to start up?

#### Error: 

Atomic renaming failure

#### Detailed Description:

During startup process HMaster does lot of initialization steps including moving data from scratch (.tmp) folder to 
data folder; also looks at WALs (Write Ahead Logs) folder to see if there are any dead region servers and so on. 
During all these situations it does a basic 'list' command on these folders. If at anytime it sees an unexpected file 
in any of these folders it will throw an exception and hence not start.  

#### Probable Cause:

Try to identify the timeline of the file creation and see if there was a process crash at around that time in region 
server logs (Contact any of the HBase crew to assist you in doing this). This helps us provide more robust mechanisms 
to avoid hitting this bug and ensure graceful process shutdowns.

#### Resolution Steps:

In such a situation try to check the call stack and see which folder might be causing problem (for instance is it WALs 
folder or .tmp folder). Then via Cloud Explorer or via hdfs commands try to locate the problem file - usually this is a 
*-renamePending.json file (a journal file used to implement Atomic Rename operation in WASB driver; due to bugs in this 
implementation such files can be left over in cases of process crash etc). Force delete this either via Cloud Explorer. 
In addition sometimes there might also be a temporary file of the nature $$$.$$$ in this location, this cannot be seen 
via cloud explorer and only via hdfs ls command. You can use hdfs command "hdfs dfs -rm /<the path>/\$\$\$.\$\$\$" to 
delete this file.  

Once this is done, HMaster should start up immediately.

- - -

#### Error:

No server address listed in hbase: meta for region xxx

#### Detailed Description:

Customer met an issue on their Linux cluster that hbase: meta table was not online. Running hbck reported that 
"hbase: meta table replicaId 0 is not found on any region". After restarting HBase, the symptom became that the hmaster 
could not initialize. In the hmaster logs, it reported that "No server address listed in hbase: meta for region 
hbase: backup <region name>".  

#### Resolution Steps:

1. Type the following on HBase shell (change actual values as applicable),  

~~~~
> scan 'hbase:meta'  
~~~~

~~~~
> delete 'hbase:meta','hbase:backup <region name>','<column name>'  
~~~~

2. Delete the entry of hbase: namespace as the same error may be reported while scan hbase: namespace table

3. Restart the active HMaster from Ambari UI to bring up HBase in running state.  

4. Run the following command on HBase shell to bring up all offline tables:

~~~~ 
hbase hbck -ignorePreCheckPermission -fixAssignments 
~~~~

#### Further Reading:

[Unable to process HBase table](http://stackoverflow.com/questions/4794092/unable-to-access-hbase-table)

- - -

#### Error:

HMaster times out with fatal exception like java.io.IOException: Timedout 300000ms waiting for namespace table to be 
assigned

#### Detailed Description:

Customer ran into this issue when apparently they had a lot of tables and regions and had not flushed when they 
restarted their HMaster services. Restart was failing with above message. No other error found.  

#### Probable Cause:

This is a known "defect" with the HMaster - general cluster startup tasks can take a long time and the HMaster shuts 
down because of the namespace table isn’t yet assigned - which is hit only in this scenario where large amount of 
unflushed data exists and a timeout of five minutes is not sufficient
  
#### Resolution Steps:

1. Access Ambari UI, go to HBase -> Configs, in custom hbase-site.xml add the following setting: 

~~~~~
Key: hbase.master.namespace.init.timeout Value: 2400000  
~~~~~

2. Restart required services (Mainly HMaster and possibly other HBase services).  

- - -

#### Error:

long Java GC pause caused regionserver reboot periodically

#### Detailed Description:

Nodes rebooted periodically. HMaster was initializing and lots of regions in transition. From the regionserver logs we could find

~~~~
2017-05-09 17:45:07,683 WARN  [JvmPauseMonitor] util.JvmPauseMonitor: Detected pause in JVM or host machine (eg GC): pause of approximately 31000ms
2017-05-09 17:45:07,683 WARN  [JvmPauseMonitor] util.JvmPauseMonitor: Detected pause in JVM or host machine (eg GC): pause of approximately 31000ms
2017-05-09 17:45:07,683 WARN  [JvmPauseMonitor] util.JvmPauseMonitor: Detected pause in JVM or host machine (eg GC): pause of approximately 31000ms
~~~~

#### Probable Cause:

The root cause was due to long regionserver JVM GC pause. This caused regionserver unresponsive and it was not able to send heart beat to HMaster within the zk session timeout (40s: although in hbase zookeeper.session.timeout is 120s, in zookeeper we did not explicitly set it, so by default zookeeper will use 40s. This has been fixed in newly created clusters. This cluster is old and I have manually set it to 120s in Ambari Zookeeper section). Thus, HMaster thought the regionserver was dead and will abort the regionserver and restart. Frequent regionserver restart may also result in other issues like atomic renaming failures.

#### Resolution Steps:

One thing to understand first is that in order to change the zookeeper session timeout, not only hbase-site setting "zookeeper.session.timeout" but also zookeeper zoo.cfg setting "maxSessionTimeout" need to be changed.

1. Access Ambari UI, go to HBase -> Configs -> Settings, in Timeouts section you can change the value of Zookeeper Session Timeout. 

2. Access Ambari UI, go to Zookeeper -> Configs -> Custom zoo.cfg, add/change the following setting. Make sure the value is the same as hbase "zookeeper.session.timeout".

~~~~~
Key: maxSessionTimeout Value: 120000  
~~~~~

2. Restart required services.

- - -

#### Error:

log splitting failure due to wrong configurations

#### Detailed Description:

Customer reported all the HMasters failed to come up on a HBase cluster. SSH to one of the zookeeper nodes and check HMaster log under /var/log/hbase, the error was log splitting failure. 

#### Probable Cause:

Looking at HDFS and HBase settings, customer tried to use a secondary storage account for HBase and there were two misconfigured settings: 

hbase.rootdir: wasb://<container>@<storageaccount>.blob.core.windows.net/ 

fs.azure.page.blob.dir: /hbase/WALs,/hbase/oldWALs,/mapreducestaging,/hbase/MasterProcWALs,/atshistory,/tezstaging,/ams/hbase 

In this way, all HBase folders were created under root. WALs and oldWALs were block blobs. This might cause log splitting failure because we had never tested block blob WAL files. 


#### Resolution Steps:

The fix is to set hbase.rootdir: wasb://<container>@<storageaccount>.blob.core.windows.net/hbase and restart services on Ambari.