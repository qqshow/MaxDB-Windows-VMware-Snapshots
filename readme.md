# Background

If you have a Windows VM running in ESX which is hosting a MaxDB database and you want to create VMware Snapshots where the database is in an Application Consistent state then we can use these scripts.   The only requirements are:

1)  VMware tools must be installed and running on the VMware VM.
2)  The VM cannot be in a Microsoft Cluster (MSCS).  This is because MSCS requires all adapters to be in bus sharing mode, which blocks VMware snapshots.
3)  The Actifio Connector does not need to be installed for snapshots to work, but may be needed to assist with co-ordinating mounts.
4)  The snapshot policy in the Template needs to specify Application Consistency or MaxDB will not be quiesced before each snapshot.

Effectively the order of events will be:

1)  Actifio requests an Application Consistent VMware snapshot of the VM 
2)  ESX requests VMware tools running in the VM to run any script located in c:\windows\pre-freeze-script.bat
3)  ESX takes a snapshot of the VM
4)  ESX requests VMware tools running in the VM to run any script located in c:\windows\post-thaw-script.bat
5)  Actifio creates an image of the VMware snapshot
6)  Actifio requests the VMware snapshot be removed

# Important points

This technique suspends the log writer, which suspends update transactions.  Therefore this procedure needs to be run as quickly as possible.  VMware snapshots are good for this because snapshot creation at the ESX level is very fast, allowing the log writer to be resumed quickly.

By suspending the log writer, no more Checkpoints (also called Savepoints) can be written.  This means the last Savepoint is used during database restart or restore.


# Supporting documentation

This kind of backup is documented in SAP Note 616814.

Recovery information is documented in SAP Note 371247.  

# Supported MaxDB Versions

MaxDB 7.3 does not support exiting the session between suspend and resume.  For this reason use version 7.4 and above.
From version 7.8.0.2 and above SAP suggest a different method documented in SAP note 1928060

# Customizing the Scripts

The two scripts need three settings customized.   The password is stored in the clear.

```
@SET DATABASE=MAXDB1
@SET USERNAME=DBADMIN
@SET PASSWORD=passw0rd
```
The script also directs logging to c:\tmp\vmtools.log so please ensure the c:\tmp folder exists or choose a different folder.

# Installing the Scripts

Once the scripts are customized place them in the following locations:
```
c:\windows\pre-freeze-script.bat
c:\windows\post-thaw-script.bat
```
Ensure the logging folder exists (C:\tmp is the default)

# Testing the Scripts

Open a command prompt using 'Run as Administrator' and run these two commands:
```
cd c:\windows
pre-freeze-script.bat
```
Now check the log file.   Expected output is as follows.
The database state should have no state.  After the freeze the state should be USR HOLD
```
------------------------------------------ 
About to freeze MAXDB1 
Date and time: Tue 01/30/2018 20:36:58.54 
***** log active state 
OK
        

SERVERDB: MAXDB1

ID   UKT  Win   TASK       APPL Current         Timeout Region     Wait 
          tid   type        pid state          priority cnt try    item 

Console command finished (2018-01-30 20:36:58).
***** issue suspend 
OK
IO SEQUENCE                    = 29664
***** log active state after suspend 
OK
        

SERVERDB: MAXDB1

ID   UKT  Win   TASK       APPL Current         Timeout Region     Wait 
          tid   type        pid state          priority cnt try    item 
T2     2  0x68C Logwr           USR HOLD (248)  2     0 0               10(s)

Console command finished (2018-01-30 20:36:59).
```

Now run this command:
```
post-thaw-script.bat
```
Now check the log file.   Expected output is as follows.
The database state should be USR HOLD.  After the resume there should be no state.

```
------------------------------------------ 
About to thaw MAXDB1 
Date and time: Tue 01/30/2018 20:37:45.69 
***** log active state  
OK
        

SERVERDB: MAXDB1

ID   UKT  Win   TASK       APPL Current         Timeout Region     Wait 
          tid   type        pid state          priority cnt try    item 
T2     2  0x68C Logwr           USR HOLD (248)  2     0 0               10(s)

Console command finished (2018-01-30 20:37:45).
***** issue resume 
OK
***** log active state after resume 
OK
        

SERVERDB: MAXDB1

ID   UKT  Win   TASK       APPL Current         Timeout Region     Wait 
          tid   type        pid state          priority cnt try    item 

Console command finished (2018-01-30 20:37:46).
```

You are now ready to begin creating Application Consistent Snapshots of this VM.

# Troubleshooting

1)  The scripts do not run - location and name

Check their file location and naming.    Some versions of ESX require the files to be in different locations
```
ESX/ESXi 3.5 Update 1 or earlier
C:\Windows\<pre-freeze-script.bat>
C:\Windows\<post-thaw-script.bat></pre-freeze-script.bat>

ESX/ESXi 3.5 Update 2 or later	
C:\Program Files\VMware\VMware Tools\backupScripts.d\

ESX/ESXi 4.x	
C:\Windows\backupScripts.d\

ESXi 5.0	
C:\Windows\
C:\Program Files\VMware\VMware Tools\backupScripts.d\

ESXi 5.1 or later
C:\Windows\<pre-freeze-script.bat>
C:\Windows\<post-thaw-script.bat></pre-freeze-script.bat>
```
2)  The scripts do not run - check SLA Template

The Scripts will only be run if an Application Consistent snapshot is requested.   If the snapshot policy requests crash consistent snapshots then the request will never be sent to VMware tools to run the scripts.

3) The scripts fail because DBMCLI.exe cannot be found

a)  Either you did not reboot after installing MAXDB so there is no path environment variable
b)  MAXDB was installed as part of SAP installation and there is no PATH to that folder

In which case determine location of dbmcli.exe and update the freeze and thaw scripts with actual location.
Could be:    
```
C:\sapdb\programs\pgm\dbmcli.exe
C:\Program Files\sdb\programs\pgm\dbmcli.exe
```

4)  Manually running the script fails with errors like this:
```
Error! Connection failed to node (local) for database MAXDB1:
-24700,ERR_DBMSRV_NOSTART: Could not start DBM server.
-24701,ERR_EXHNDLR: Could not initialize exception handler.
-24748,ERR_FILEOPEN: Error opening file C:\ProgramData\sdb\data\wrk\dbmsrv_SGWINDOWS001.err
-24826,ERR_NIERROR: Can not open file 'C:\ProgramData\sdb\data\wrk\dbmsrv_SGWINDOWS001.err'. (System error 5; Access is denied.)
```
You need to start the Command Prompt using 'Run as Administrator'

5)  The VMware snapshots fail with a message that talks about bus sharing.

This VM is part of a Microsoft Cluster and cannot be protected using this method.

