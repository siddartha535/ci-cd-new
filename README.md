Steps for Migration from MQ V8 to MQ V9 on SLES12
Prepare a similar PPT for the MQ v9 migration and should be included in the Announcement. 
https://team.sp.wp.corpintra.net/sites/00220/fachteams/message_queuing/Actions/ITI-EA%20IBM%20MQ%209%20Migrationplan.pptx 
2) Raise a CISM Change ticket.
3) Have one excel sheet for all the server and Qmgr details.

Preparation Steps for Day-before:
Check for the file system space for /var/mw/mqbackup directory do the cleanup if required.
df -h /var/mw/mqbackup
Instructions On Implementation Day:
Step: 1 (MQ Ops Team: Back up)

Take a full backup of queue manager and place the tar file on /var/mw/mqbackup
Run the below scripts to take the backup on the source server:

As root: Execute the below command and enter the queue manager name for which backup are required

dspmq -o installation;echo "enter queue managers separated by space: ";read qmgrs;echo "chosen: $qmgrs";
Channel & Queue Status Check:

for qmgr in ${qmgrs};do . mqsetenv ${qmgr}; [[ -d /var/mw/mqbackup/${qmgr} ]] ||  mkdir /var/mw/mqbackup/${qmgr};chown -R mqm:mqm /var/mw/mqbackup/${qmgr}; su mqm -c"ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";sleep 10;echo "display chstatus(*) where (STATUS NE RUNNING)" |su mqm -c "runmqsc ${qmgr}" >/home/mqm/${qmgr}_retrying_chstatus.log; echo "display qs(*) where (CURDEPTH GT 0)" | su  mqm -c "runmqsc ${qmgr}";cat /home/mqm/${qmgr}_retrying_chstatus.log; done
	
🡪 Make a note on the channels which are not in running state and save and place it under /home/mqm/{qmgrs}_retrying_chstatus.log 

Takes a backup of the current qmgr status
/opt/mw/bin/mw_backup.sh 

Note: This is an known error. 
Copy '/etc/mw/mqm/*.bip' to '/var/mw/backup/current/mqm' ...
  Failed to find logical volume "data1vg/lvvar_mw_qbackup" 
expr: syntax error

Disable the channels and automatic listener start:

for qmgr in ${qmgrs}; do /opt/mw/mqm/bin/mqstopchannels.sh ${qmgr} ALL; 
echo "alter listener (LISTENER.DEFAULT.01) TRPTYPE(TCP) CONTROL(MANUAL)" | su - mqm -c ". mqsetenv ${qmgr} ;runmqsc ${qmgr}";done

Execute the below command to clean up the transactions logs.
/bin/su - mqm -c '/opt/mw/mqm/bin/mqlogcleanup.sh -all'
    
Note:  If we see error as 
 /opt/mw/mqm/bin/mqlogcleanup.sh[129]: .[101]: checkAndInvoke[53]: .: /etc/mw/mqm/locals_<Queue Manger>: cannot open [Permission denied]as cannot open [Permission denied]  
 Please change the ownership of /etc/mw/mqm/locals_<Queue Manger>  files to mqm:mqm 

Execute the below command to stop the queue manager
for qmgr in ${qmgrs}; do /opt/mw/mqm/bin/mqstop.sh ${qmgr};done


We will start the backup of the objects and tar it and send the file to backup directory.
echo "Start Backup of queue managers $qmgrs";for qmgr in $qmgrs;do bfile=MQmigration-${qmgr}-MQv8-to-MQv9.tar.gz;tar -cvzf /var/mw/mqbackup/${bfile} /var/mqm/qmgrs/${qmgr} /var/mqm/log/${qmgr};[[ $? -eq 0 ]] || break; echo "  processed ${qmgr}, backup in /var/mw/mqbackup/${bfile}";done;echo "Backup done, files saved in /var/mw/mqbackup/"


Step: 2 Migrate from v8.0.0.6 to 9.0.0.2	

Switching queue managers from MQv8 to MQv9
dspmq;echo "enter queue managers separated by space: ";echo "Selected queue manager: ${qmgrs}"; read qmgrs;echo "chosen: $qmgrs";
for qmgr in $qmgrs;do su - mqm -c ". mqsetenv $qmgr; /opt/mqm900/bin/setmqm -m $qmgr -n $(/opt/mqm900/bin/dspmqver -bf 512)";done; 
Verify INSTNAME(Installation1) INSTPATH(/opt/mqm800) INSTVER(8.0.0.6) using below command 
 dspmq -o installation

Make the Installation2 as Primary Installation:
/opt/mqm900/bin/setmqinst -x -n Installation1 
/opt/mqm900/bin/setmqinst -i -n Installation2 

dspmqver -i

Start the Queue managers

for qmgr in $qmgrs ; do /opt/mw/mqm/bin/mqstart.sh ${qmgr};done

Perform Check

/opt/mw/mqm/bin/mqcheckafteraction.sh

Make the Control of the Listener to QMGR
for qmgr in ${qmgrs};do . mqsetenv ${qmgr};echo "alter listener (LISTENER.DEFAULT.01) TRPTYPE(TCP) CONTROL(QMGR)" | su mqm -c "runmqsc ${qmgr}";done

Restart of the Queue managers
for qmgr in ${qmgrs};do /opt/mw/mqm/bin/mqstop.sh ${qmgr}; /opt/mw/mqm/bin/mqstart.sh ${qmgr}; done

Check for Channel & Queue status:
for qmgr in ${qmgrs};do . mqsetenv ${qmgr};su mqm -c "ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";sleep 10;echo "display chstatus(*) where (STATUS NE RUNNING)" |su mqm -c "runmqsc ${qmgr}"; echo "display qs(*) where (CURDEPTH GT 0)" |su mqm -c "runmqsc ${qmgr}";done
Stop and Reboot
for qmgr in ${qmgrs};do /opt/mw/mqm/bin/mqstop.sh ${qmgr};done
Reboot

Step: 4 (MQ Ops Team: Final Checks)
Make sure all the qmgrs are up and running state.


/opt/mw/mqm/bin/mqcheck.sh –all
/opt/mw/mqm/bin/mqcheckafteraction.sh
Perform MQ Admin connect Test
Using the MQ Resource Manager http://mqadmin.de050.corpintra.net/resource  - Change “Version” of migrated QMGRs to “9.0.0”
Make the final announcement to the Application Team to perform a connectivity check, once the patch is completed from their end as well.
Check for the alerts in HP open viewer monitoring tool.
Check for the alerts in Status Console.









Roll Back Plan (MQv9 to MQv8)

Execute as Root
dspmq;echo "enter queue managers separated by space: ";read qmgrs;echo "chosen: $qmgrs";echo "check ${migrationDir}=${backupDir}=/var/mqbackup"

Channel & Queue Status Check:
for qmgr in ${qmgrs};do . mqsetenv ${qmgr}; [[ -d /var/mw/mqbackup/${qmgr} ]] ||  mkdir /var/mw/mqbackup/${qmgr};chown mqm:mqm ${qmgr}; su - mqm -c" echo "display chstatus(*) where (STATUS NE RUNNING)" |su - mqm -c "runmqsc ${qmgr}" >/var/mw/mqbackup/${qmgr}_chl_chstatus.log; echo "display qs(*) where (CURDEPTH GT 0)" | su - mqm -c "runmqsc ${qmgr}";cat /var/mw/mqbackup/${qmgr}_ chl_chstatus.log; done 


Disable the channels and automatic listener start:

for qmgr in ${qmgrs}; do /opt/mw/mqm/bin/mqstopchannels.sh ${qmgr}; 
echo "alter listener (LISTENER.DEFAULT.01) TRPTYPE(TCP) CONTROL(MANUAL)" | su - mqm -c ". mqsetenv ${qmgr} ;runmqsc ${qmgr}";done

Save channel Sequence-number 

mqm:/> for qmgr in ${qmgrs}; do . mqsetenv ${qmgr}; su - mqm -c" echo "DISPLAY CHSTATUS(*) SAVED CURSEQNO LSTSEQNO where (LSTSEQNO NE 0)"| runmqsc ${qmgr} > ${backupDir}/${qmgr}/${qmgr}.seqnr ;done


Take dump of messages present on Local queue leaving XMITQ’s
for qmgr in ${qmgrs};do /opt/mw/mqm/bin/mqsaveallmsgs.sh ${qmgr};done
Stop the queue manager
for qmgr in ${qmgrs};do /opt/mw/mqm/bin/mqstop.sh ${qmgr};done
Delete the queue manager using dltmqm command
for qmgr in ${qmgrs}; do /opt/mqm900/bin/dltmqm ${qmgr} force; done


Restore backup which was taken for MQv8

cp -p /var/mw/backup/current/mqm/mqs.ini /var/mqm

If restoring one queue manager get the details of that particular queue manager from archive directory from file mqs.ini and make entries to existing mqs.ini file.
for qmgr in ${qmgrs};do (cd /;tar -xvzf  /var/mw/mqbackup/MQmigration-${qmgr}-MQv8-to-MQv9.tar.gz); done

Start the queue managers 
for qmgr in ${qmgrs};do /opt/mw/mqm/bin/mqstart.sh ${qmgr};done

Check values of INSTNAME, INSTPATH and INSTVER whether it has changed accordingly to MQV8

dspmq –o all

Make the Control of the Listener to QMGR
for qmgr in ${qmgrs};do . mqsetenv ${qmgr};echo "alter listener (LISTENER.DEFAULT.01) TRPTYPE(TCP) CONTROL(QMGR)" | su mqm -c "runmqsc ${qmgr}";done

/opt/mw/mqm/bin/mqstop.sh –all

Reboot 

(MQ Ops Team: Post-Checks)
/opt/mw/mqm/bin/mqstart.sh –all

Check for Channel & Queue status:

dspmq;echo "enter queue managers separated by space: ";read qmgrs;echo "chosen: $qmgrs";

for qmgr in ${qmgrs};do . mqsetenv ${qmgr};su mqm -c "ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";sleep 10;echo "display chstatus(*) where (STATUS NE RUNNING)" |su mqm -c "runmqsc ${qmgr}"; echo "display qs(*) where (CURDEPTH GT 0)" |su mqm -c "runmqsc ${qmgr}";done

(MQ Ops Team: Final Checks)
Make sure all the qmgrs are up and running state.
/opt/mw/mqm/bin/mqcheck.sh –all
/opt/mw/mqm/bin/mqcheckafteraction.sh
Perform MQ Admin connect Test
Using the MQ Resource Manager http://mqadmin.de050.corpintra.net/resource  - Change “Version” of migrated QMGRs to “9.0.0”
Make the final announcement to the Application Team to perform a connectivity check, once the patch is completed from their end as well.
Check for the alerts in HP open viewer monitoring tool.
Check for the alerts in Status Console.


MQ Logparser script:
Script is placed under /opt/bu_master/opt/hosts/<Server Name>/everynight.d/
Script File Name: N501_Logparser_Check
Monitor the log file: Please verify the log next day /var/opt/bu/everynight.<Day>.log & /var/opt/bu/everynight.<Day>.err files to confirm if the job had run overnight.
For more information, please refer the document. Link











