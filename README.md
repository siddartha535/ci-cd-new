
Release Migration



Migration MQ7 to MQ8
Migration Tasks
Prerequisites

Make sure to run all commands in migration scenario as root
xxx:/> sudo su -

Check for enough space for Backups in /var/mw/mqbackup (80% free):
root:/> df -h | grep backup

 Checks:
Run check scripts
root:/> /opt/mw/mqm/bin/mqcheck.sh -all 
Expected output:

INFO    : 2015-04-08#11:04:46 - Processing queue manager XBO4003I
INFO    : MQSeries 800                      04/08/15 11:04:46
INFO    : Queue Manager:     XBO4003I
INFO    : OK - all necessary queue manager kernel processes are up and running
INFO    : commandserver: running
INFO    : channelinitiator: running
INFO    : Check Listener
   LISTENER(LISTENER.ADMIN.01)             STATUS(RUNNING)
INFO    : LISTENER.ADMIN.01 processes are up and running
   LISTENER(LISTENER.DEFAULT.01)           STATUS(RUNNING)
INFO    : LISTENER.DEFAULT.01 processes are up and running
    is running   :
LISTENER.ADMIN.01
LISTENER.DEFAULT.01
   must running :
LISTENER.ADMIN.01
LISTENER.DEFAULT.01
INFO    : Check Trigger
   SERVICE(TRIGGER.SYSTEM)                 STATUS(RUNNING)
INFO    : TRIGGER.SYSTEM processes are up and running
    is running   :
TRIGGER.SYSTEM
   must running :
…

root:/> /opt/mw/mqm/bin/mqcheckqmgr.sh -all
expected output:

root:~ # /opt/mw/mqm/bin/mqcheckqmgr.sh -all

INFO    : 2015-04-08#11:07:11 - Processing queue manager XBO4001I
INFO    : MQSeries  800           04/08/15 11:07:11
INFO    : QMGR-Name:    XBO4001I
INFO    : OK - all necessary queue manager kernel processes are up and running
INFO    : commandserver
           is up and running
INFO    : channelinitiator is running
…

root:/> /opt/mw/install/mqserver-check.sh
expected output:

root:~ # /opt/mw/install/mqserver-check.sh
=================================================
check the ILSE mq installation on 2015-04-08#11-08
with /opt/mw/install/mqserver-check.sh
location: i050almqs171 Linux 3.0.101-0.47.52-default x86_64
=================================================
              check mq-Version
=================================================
Name:        WebSphere MQ
Version:     7.0.1.12
CMVC level:  p701-112-140319
BuildType:   IKAP - (Production)
=================================================
                check ILMT
=================================================
check the /etc/tlmagent.ini
scan_group = ITI_EA050_NONPROD
server = lmt-server.de050.corpintra.net
=================================================
              check directory
=================================================
OK      BackUpDirectory '/var/mw/mqbackup' exist
=================================================
                  check FS
=================================================

  PV /dev/sdd    VG data2vg   lvm2 [450.00 GiB / 50.85 GiB free]
  PV /dev/sdc    VG data1vg   lvm2 [150.00 GiB / 30.37 GiB free]
  PV /dev/sdb1   VG vgswap    lvm2 [5.97 GiB / 64.00 MiB free]
  PV /dev/sda2   VG vg00      lvm2 [19.72 GiB / 3.25 GiB free]
  Total: 4 [625.68 GiB] / in use: 4 [625.68 GiB] / in no VG: 0 [0   ]
=================================================
          Check installed packages
=================================================
OK       found 'mqserver-cg-unix-8.0.0.1-1_1'
OK       found 'mw-base-1.2.7-1'
OK       found 'cgmq-wmq-8.0.0.2_2-1'
OK       found 'mqtools-cg-unix-8.0.0_3-1'
=================================================
                     TSM Check
-------------------------------------------------
OK       TSM are ready for work
=================================================
              HP OpenView Check
=================================================
 5443  5443  5443 ?        00:02:06   ovcd

OK       HPOpenView main process 'ovcd' found
=================================================
       Check/Update Repository CGMQ
=================================================
# | Alias                                       | Name          | Enabled | Refresh | Type
--+---------------------------------------------+---------------+---------+---------+-------
1 | spacewalk                                   | spacewalk     | Yes     | Yes     | plugin
2 | CGMQ                                        | CGMQ          | No      | Yes     | rpm-md
3 | CGMQ70_DEV                                  | CGMQ70_DEV    | Yes     | Yes     | rpm-md
4 | http-itiearepo.de050.corpintra.net-c09a3f95 | private-itiea | Yes     | Yes     | rpm-md
OK       Repository CGMQ disabled
=================================================
                 END CHECK
=================================================

Start sender channels
root:/> OLDIFS=${IFS};IFS=$'\n';for line in `dspmq`; do echo "`expr substr $line 8 8`";done;echo "ALL";echo "Enter QMGR names seperated by a whitespace, 'ALL' for all QMGRs:";read qmgrs;if [[ $qmgrs == "ALL" ]];then qmgrs="";for line in `dspmq`; do qmgrname=`expr substr $line 8 8`;qmgrs="$qmgrs $qmgrname";done;fi;echo "Selected qmgrs: $qmgrs";IFS=${OLDIFS};
root:/> for qmgr in ${qmgrs};do su mqm -c"ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";done

output:

Working with Queue Manager : EKE2001I and channel type : SDR
Excluding channels with 'SYSTEM.*' in the name
Use migration directory /var/mw/mqbackup
Warning: AMQ9514: Channel 'EKE2001I.XBO4001I.OS' is in use. for Channel : CHANNEL(EKE2001I.XBO4001I.OS) ; continuing

Check status, needed for comparison after migration! Save channel names not running – check why!
root:/> for qmgr in ${qmgrs};do su mqm -c"echo 'display chstatus(*) where (STATUS NE RUNNING)' | runmqsc ${qmgr} | grep CHANNEL";done

expected output: nothing

Check MQ Admin connection
Log in to MQAdmin
Menu: Admin – choose queue manager 
if connection is available, there are values of the queue manager shown
Menu: Site – Agents
The agents column “available” has to be “true”

Check FDC errors:
root:/> ls -ltrh /var/mqm/errors
root:/> for qmgr in ${qmgrs};do ls -ltrh /var/mqm/qmgrs/${qmgr}/errors ;done

Run log cleanup and saveqmgr - scripts
root:/> /bin/su - mqm -c '/opt/mw/mqm/bin/mqlogcleanup.sh -all'
root:/> /bin/su - mqm -c '/opt/mw/mqm/bin/mqsaveqmgr.sh -all'

Backup-Check:
root:/> suchstr="nsrexecd";erg_legato=$(ps -efww | grep ${suchstr} | grep -v grep | wc -l);if [[ "${erg_legato}" -gt "0" ]];then echo "Legato Check OK";erg_legato2=`echo versions /var/mqm/mqs.ini | recover 2>&1 | grep mqs.ini | wc -l`;if [[ "${erg_legato2}" -gt "1" ]];then echo "  Legato Recover Test OK";else echo "  Legato Recover Test ERROR";fi;fi;dsmc q fi > /dev/null 2>&1 ;erg_tsm=$?;if [[ "${erg_tsm}" -eq "0" || "${erg_tsm}" -eq "8" ]];then echo "TSM Check OK";fi;if [[ -f /usr/openv/netbackup/bin/bplist ]];then erg_NetBackUp=`/usr/openv/netbackup/bin/bplist -l /var/mqm/mqs.ini | wc -l`;if [[ "${erg_NetBackUp}" -gt "1" ]];then echo "NetBackup Check OK";fi;fi;

In case of “TSM Check OK” run TSM backup:
root:/> dsmc incr

In case of “NetBackup Check OK” run NetBackup:
root:/> /usr/openv/netbackup/bin/bpbackup -L /tmp/bkup-migMQ8-var.log  /var
root:/> /usr/openv/netbackup/bin/bpbackup -L /tmp/bkup-migMQ8-etc.log  /etc

In case of “Legato Check OK” and “Legato Recover Test ERROR”:
daily nightly Legato Backups. Check with:
root:/> /opt/bu/tools/is_on_tape –d /opt
for existing backups.

In case of “Legato Check OK” and “Legato Recover Test OK” run Legato Backup:
root:/> save /opt; save /var
	On Error: Check point iii.

Release Software for environment, if not done yet!
TODO: Prüfschritt?

Connect to cgmq repository server using putty
cgmqrepo.de050.corpintra.net

Run commands as root:
xxx:/> sudo su -

Run script /var/cgmqrepo/release.sh depending on the environment:
For Development:
root:/> /var/cgmqrepo/release.sh DEV exec
For Integration:
root:/> /var/cgmqrepo/release.sh INT exec
For Production:
root:/> /var/cgmqrepo/release.sh PROD exec
Follow instructions of script called!


Installation on Linux
Make sure to run all commands in migration scenario as root
xxx:/> sudo su -

NEW INSTALLATION MQ8 / MQ Tools Version 8
Set repository path, ask for environment
root:/> repo=http://cgmqrepo.de050.corpintra.net/rpms
root:/> echo -n "Please enter desired environment ( TEST / DEV / INT / PROD ): ";read repoEnv
	on Chinese server: use repo-proxy:
root:/> repo=http://s104slrep785/cgmqrepo/rpms
root:/> echo -n "Please enter desired environment ( TEST / DEV / INT / PROD ): ";read repoEnv

Add an Alias for MQ v8 repository to the repository list, update rpms, refresh zypper to accept the signing key
root:/> zypper ar -f ${repo}/MQ800_${repoEnv} CGMQ800_${repoEnv};zypper -n up -r CGMQ800_${repoEnv};zypper ref
Show new Repository
root:/> zypper lr | grep CGMQ800
# 3 | CGMQ800_${repoEnv}     | CGMQ800_${repoEnv}  | Yes     | Yes

Update the MQ Tools to latest level
root:/> /opt/mw/install/mqserver-refresh.sh

Start MQ Installation
root:/> /opt/mw/install/installMQ.sh -version 800


	Press Enter after seeing the following information:
	……
System Settings
  file-max            2784 of 524288 files       (0%)    IBM>=524288       PASS

CurrentUser Limits (mqm)
  nofile       (-Hn)  10240 files                        IBM>=10240        PASS
  nofile       (-Sn)  10240 files                        IBM>=10240        PASS
  nproc        (-Hu)  31 of 4096 processes       (0%)    IBM>=4096         PASS
  nproc        (-Su)  31 of 4096 processes       (0%)    IBM>=4096         PASS


mqconfig: V3.7 analyzing SUSE Linux Enterprise Server 11 (x86_64) settings
          for WebSphere MQ V8.0

System V Semaphores
  semmsl     (sem:1)  500 semaphores                     IBM>=32           PASS
  semmns     (sem:2)  1325 of 256000 semaphores  (0%)    IBM>=4096         PASS
  semopm     (sem:3)  250 operations                     IBM>=32           PASS
  semmni     (sem:4)  38 of 1024 sets            (3%)    IBM>=128          PASS

System V Shared Memory
  shmmax              18446744073709551615 bytes         IBM>=268435456    PASS
  shmmni              53 of 4096 sets            (1%)    IBM>=4096         PASS
  shmall              20540 of 2097152 pages     (0%)    IBM>=2097152      PASS

System Settings
  file-max            2336 of 524288 files       (0%)    IBM>=524288       PASS

Current User Limits (root)
  nofile       (-Hn)  8192 files                         IBM>=10240        WARN
  nofile       (-Sn)  1024 files                         IBM>=10240        FAIL
  nproc        (-Hu)  17 of 13396 processes      (0%)    IBM>=4096         PASS
  nproc        (-Su)  17 of 13396 processes      (0%)    IBM>=4096         PASS

Check of MQ version (post Installation)
root:/> /opt/mqm800/bin/dspmqver

expected output:

Name:        WebSphere MQ
Version:     8.0.0.2
Level:       p800-002-150217.2
BuildType:   IKAP - (Production)
Platform:    WebSphere MQ for Linux (x86-64 platform)
Mode:        64-bit
O/S:         Linux 3.0.101-0.47.52-default
InstName:    Installation1
InstDesc:
Primary:     No
InstPath:    /opt/mqm800
DataPath:    /var/mqm
MaxCmdLevel: 801
LicenseType: Production

Note there are a number (1) of other installations, use the '-i' parameter to display them.

Final Check (Day before Migration)

Checks before migration:

Check for enough space for Backups in /var/mw/mqbackup:
root:/> df -h | grep backup

Run mqcheck script:
root:/> /opt/mw/mqm/bin/mqcheck.sh -all
Expected output:

root:~ # /opt/mw/mqm/bin/mqcheck.sh -all

INFO    : 2015-04-08#11:04:15 - Processing queue manager XBO4001I
INFO    : MQSeries 701                      04/08/15 11:04:15
INFO    : Queue Manager:     XBO4001I
INFO    : OK - all necessary queue manager kernel processes are up and running
INFO    : commandserver: running
INFO    : channelinitiator: running
INFO    : Check Listener
    is running   :

   must running :

INFO    : Check Trigger
    is running   :

   must running :

INFO    : queue manager status:     Running
INFO    : MQ-Agents:   XBO4001I
INFO    : mqagentadmin: OK - MQ-Admin-Agent processes is up and running
INFO    : mqagentmonitor: OK - MQ-Admin-Agent processes is up and running
INFO    : Check write/read a message
INFO    : Error using q command, trying with sample programs amqsget and amqsput
INFO    : Message write/read sucessfull.

INFO    : 2015-04-08#11:04:46 - Processing queue manager XBO4003I
INFO    : MQSeries 701                      04/08/15 11:04:46
INFO    : Queue Manager:     XBO4003I
INFO    : OK - all necessary queue manager kernel processes are up and running
INFO    : commandserver: running
INFO    : channelinitiator: running
INFO    : Check Listener
   LISTENER(LISTENER.ADMIN.01)             STATUS(RUNNING)
INFO    : LISTENER.ADMIN.01 processes are up and running
   LISTENER(LISTENER.DEFAULT.01)           STATUS(RUNNING)
INFO    : LISTENER.DEFAULT.01 processes are up and running
    is running   :
LISTENER.ADMIN.01
LISTENER.DEFAULT.01
   must running :
LISTENER.ADMIN.01
LISTENER.DEFAULT.01
INFO    : Check Trigger
   SERVICE(TRIGGER.SYSTEM)                 STATUS(RUNNING)
INFO    : TRIGGER.SYSTEM processes are up and running
    is running   :
TRIGGER.SYSTEM
   must running :
…
Run mqcheckqmgr script
root:/> /opt/mw/mqm/bin/mqcheckqmgr.sh -all
expected output:

root:~ # /opt/mw/mqm/bin/mqcheckqmgr.sh -all

INFO    : 2015-04-08#11:07:11 - Processing queue manager XBO4001I
INFO    : MQSeries  701           04/08/15 11:07:11
INFO    : QMGR-Name:    XBO4001I
INFO    : OK - all necessary queue manager kernel processes are up and running
INFO    : commandserver
           is up and running
INFO    : channelinitiator is running
…
Run mqserver-check script
root:/> /opt/mw/install/mqserver-check.sh
expected output:

root:~ # /opt/mw/install/mqserver-check.sh
=================================================
check the ILSE mq installation on 2015-04-08#11-08
with /opt/mw/install/mqserver-check.sh
location: i050almqs171 Linux 3.0.101-0.47.52-default x86_64
=================================================
              check MQ-Version(s)
=================================================
Name:        WebSphere MQ
Version:     8.0.0.2
Level:       p800-002-150217.2
BuildType:   IKAP - (Production)
Platform:    WebSphere MQ for Linux (x86-64 platform)
Mode:        64-bit
O/S:         Linux 3.0.101-0.47.52-default
InstName:    Installation1
InstDesc:
Primary:     No
InstPath:    /opt/mqm800
DataPath:    /var/mqm
MaxCmdLevel: 801
LicenseType: Production

Name:        WebSphere MQ
Version:     7.0.1.12
InstName:    Installation0
InstDesc:    IBM WebSphere MQ Installation
InstPath:    /opt/mqm
Primary:     Yes 
=================================================
                check ILMT
=================================================
check the /etc/tlmagent.ini
scan_group = ITI_EA050_NONPROD
server = lmt-server.de050.corpintra.net
=================================================
              check directory
=================================================
OK      BackUpDirectory '/var/mw/mqbackup' exist
=================================================
                  check FS
=================================================

  PV /dev/sdd    VG data2vg   lvm2 [450.00 GiB / 50.85 GiB free]
  PV /dev/sdc    VG data1vg   lvm2 [150.00 GiB / 30.37 GiB free]
  PV /dev/sdb1   VG vgswap    lvm2 [5.97 GiB / 64.00 MiB free]
  PV /dev/sda2   VG vg00      lvm2 [19.72 GiB / 3.25 GiB free]
  Total: 4 [625.68 GiB] / in use: 4 [625.68 GiB] / in no VG: 0 [0   ]
=================================================
          Check installed packages
=================================================
OK       found 'mqserver-cg-unix-8.0.0.1-1_1'
OK       found 'mw-base-1.2.7-1'
OK       found 'cgmq-wmq-8.0.0.2_2-1'
OK       found 'mqtools-cg-unix-8.0.0_3-1'
=================================================
                     TSM Check
-------------------------------------------------
OK       TSM are ready for work
=================================================
              HP OpenView Check
=================================================
 5443  5443  5443 ?        00:02:06   ovcd

OK       HPOpenView main process 'ovcd' found
=================================================
       Check/Update Repository CGMQ
=================================================
# | Alias       | Name        | Enabled | Refresh | Type
--+-------------+-------------+---------+---------+-------
1 | spacewalk   | spacewalk   | Yes     | Yes     | plugin
2 | CGMQ        | CGMQ        | No      | Yes     | rpm-md
3 | CGMQ70_INT  | CGMQ70_INT  | Yes     | No      | rpm-md
4 | CGMQ800_INT | CGMQ800_INT | Yes     | Yes     | rpm-md
5 | itiea       | itiea       | Yes     | No      | rpm-md

OK       Repository CGMQ disabled
=================================================
                 END CHECK
=================================================
Run mqcheckafteraction.sh script
dwoealmqs240:~ # /opt/mw/mqm/bin/mqcheckafteraction.sh
Hostname dwoealmqs240
Check for not running qmgrs
Check Current Aug 05 error logs or FDCs
Check qmgrs (one minute for each qmgr)
ERROR   : mqagentadmin:  The agent processes is not active on this server !!!
ERROR   : mqagentadmin: not OK
ERROR   : ===>  Please check !!!
ERROR   : mqagentmonitor:  The agent processes is not active on this server !!!
ERROR   : mqagentmonitor: not OK
ERROR   : ===>  Please check !!!
Check FS with thresholds  -  logs >35% or other >70%
End of script

Start sender channels
root:/> OLDIFS=${IFS};IFS=$'\n';for line in `dspmq`; do echo "`expr substr $line 8 8`";done;echo "ALL";echo "Enter QMGR names seperated by a whitespace, 'ALL' for all QMGRs:";read qmgrs;if [[ $qmgrs == "ALL" ]];then qmgrs="";for line in `dspmq`; do qmgrname=`expr substr $line 8 8`;qmgrs="$qmgrs $qmgrname";done;fi;echo "Selected qmgrs: $qmgrs";IFS=${OLDIFS};
root:/> for qmgr in ${qmgrs};do su mqm -C"ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";done

	expected output:

	Working with Queue Manager : EKE2001I and channel type : SDR
	Excluding channels with 'SYSTEM.*' in the name
	Use migration directory /var/mw/mqbackup
	Warning: AMQ9514: Channel 'EKE2001I.XBO4001I.OS' is in use. For channel : CHANNEL(EKE2001I.XBO4001I.OS) ; continuing

Check status, needed for comparison after migration! Save channel names not running – check why!
root:/> for qmgr in ${qmgrs};do su mqm -C"echo 'display chstatus(*) where (STATUS NE RUNNING)' | runmqsc ${qmgr} | grep CHANNEL";done
	expected output: nothing

Check MQ Admin connection
root:/> dspmq
	- Log in to MQAdmin
	- Menu: Admin -> choose queue manager 
	-> if connection is available, there are values of the queue manager shown
	- Menu: Site -> Agents
	-> The agents column "available" has to be "true"

Check FDC errors:
root:/> ls -ltrh /var/mqm/errors
root:/> for qmgr in ${qmgrs};do ls -ltrh /var/mqm/qmgrs/${qmgr}/errors ;done

Run log cleanup and saveqmgr scripts
root:/> /bin/su - mqm -c '/opt/mw/mqm/bin/mqlogcleanup.sh -all'
root:/> /bin/su - mqm -c '/opt/mw/mqm/bin/mqsaveqmgr.sh -all'

Check for MQ8 installation
root:/> /opt/mqm800/bin/dspmqver -f 2
Version:     8.0.0.2
Check for MQ7 installation
root:/> dspmqver -f 2
Version:     7.0.1.12


Migration of queue managers: 7 -> 8

Make sure to run all commands in migration scenario as root
xxx:/> echo -ne "\033]0;$(hostname)\007";sudo su -

Checks before migration:
Start sender channels
root:/> dspmq;echo "enter queue managers separated by space: ";read qmgrs;echo "chosen: $qmgrs";
root:/> for qmgr in ${qmgrs};do su mqm -c"ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";sleep 10;echo "display chstatus(*) where (STATUS NE RUNNING)" | su - mqm -c "runmqsc ${qmgr}";done

	expected output:

	Working with Queue Manager : EKE2001I and channel type : SDR
	Excluding channels with 'SYSTEM.*' in the name
	Use migration directory /var/mw/mqbackup
	Warning: AMQ9514: Channel 'EKE2001I.XBO4001I.OS' is in use. For channel : CHANNEL(EKE2001I.XBO4001I.OS) ; continuing


Run log cleanup and middleware backup
root:/> /bin/su - mqm -c '/opt/mw/mqm/bin/mqlogcleanup.sh -all'
root:/> /opt/mw/bin/mw_backup.sh

Read queue managers to stop for migration and backup:
root:/> OLDIFS=${IFS};IFS=$'\n';for line in `/opt/mqm800/bin/dspmq -o installation|grep /opt/mqm\)`; do echo "`expr substr $line 8 8`";done;echo "ALL";echo "Enter QMGR names to stop seperated by a whitespace, 'ALL' for all QMGRs:";read qmgrnames;if [[ $qmgrnames == "ALL" ]];then qmgrnames="";for line in `/opt/mqm800/bin/dspmq -o installation|grep /opt/mqm\)`; do qmgrname=`expr substr $line 8 8`;qmgrnames="$qmgrnames $qmgrname";done;fi;echo "Selected qmgrs: $qmgrnames";IFS=${OLDIFS};

Stop queue managers:
root:/> for name in $qmgrnames; do mqstop $name;done
	
Data Backup:
Check which backup system is installed:
root:/> suchstr="nsrexecd";erg_legato=$(ps -efww | grep ${suchstr} | grep -v grep | wc -l);if [[ "${erg_legato}" -gt "0" ]];then echo "Legato Check OK";erg_legato2=`echo versions /var/mqm/mqs.ini | recover 2>&1 | grep mqs.ini | wc -l`;if [[ "${erg_legato2}" -gt "1" ]];then echo "  Legato Recover Test OK";else echo "  Legato Recover Test ERROR";fi;fi;dsmc q fi > /dev/null 2>&1 ;erg_tsm=$?;if [[ "${erg_tsm}" -eq "0" || "${erg_tsm}" -eq "8" ]];then echo "TSM Check OK";fi;if [[ -f /usr/openv/netbackup/bin/bplist ]];then erg_NetBackUp=`/usr/openv/netbackup/bin/bplist -l /var/mqm/mqs.ini | wc -l`;if [[ "${erg_NetBackUp}" -gt "1" ]];then echo "NetBackup Check OK";fi;fi;

In case of “TSM Check OK” run TSM backup:
root:/> dsmc incr

In case of “NetBackup Check OK” run NetBackup:
root:/> bpbackup -L /tmp/bkup-migMQ8-var.log  /var
root:/> bpbackup -L /tmp/bkup-migMQ8-etc.log  /etc

In case of “Legato Check OK” AND “Legato Recover Test ERROR” NO MANUAL DATA BACKUP POSSIBLE – check for last nightly backup:
root:/> /opt/bu/tools/is_on_tape -d /opt/mw

In case of “Legato Check OK” and “Legato Recover Test OK” run Legato Backup: IN HAMBACH: nightly backup – save not possible!
root:/> save /var /etc

Run delete operations (all log files, error log files, trace files, old archived log files):
root:/> for name in $qmgrnames;do echo "deletion of log files ($name): ";find /var/mqm/qmgrs/ -iregex ".*${name}.*AMQERR0.*\.LOG" -delete;echo "deletion of old log files ($name): ";find /var/mqm/log  -iregex ".*${name}.*\.gz" -delete;done;echo "deletion of error log files: ";rm /var/mqm/errors/*;echo "deletion of trace files: ";rm /var/mqm/trace/AMQ*;

Save /var/mqm/qmgrs/<qmgr-name> in /var/mq/mqbackup:
root:/> echo "Start Backup of queue managers $qmgrnames";for name in $qmgrnames;do bfile=MQ8migration-$name-MQ7-to-MQ8-`(date +"%Y-%m-%d#%H:%M:%S")`.tar.gz;tar -cvzf /var/mw/mqbackup/${bfile} /var/mqm/qmgrs/$name /var/mqm/log/$name;echo "  processed ${name}, backup in /var/mw/mqbackup/${bfile}";done;echo "Backup done, files saved in /var/mw/mqbackup/"

Asure usability of SSLv3 protocol in MQ8:
root:/> echo "Start adding qm.ini setting for SSLv3 protocol of queue managers $qmgrnames";for name in $qmgrnames;do sed -i '/SSL:/a \ \ \ AllowSSLV3=Y' /var/mqm/qmgrs/${name}/qm.ini;echo "  processed queue manager $name";done;cat /var/mqm/qmgrs/${name}/qm.ini;

Change Installation of QMGRs to MQ8 Installation:
root:/> for name in $qmgrnames;do su - mqm -c "/opt/mqm800/bin/setmqm -m $name -n $(/opt/mqm800/bin/dspmqver -bf 512)";done
#The setmqm command completed successfully.

root:/> /opt/mqm800/bin/dspmq -o installation


Start queue managers using MQ8
root:/> for name in $qmgrnames;do mqstart $name;done;/opt/mqm800/bin/dspmq
Using the MQ Resource Manager http://mqadmin.de050.corpintra.net/resource - Change “Version” of migrated QMGRs to “8.0.0”:

root:/> echo $qmgrnames

Menu: Queuemanagers 
<crtl> - <f> enter QMGR name (find QMGR in Browser)
Klick on QMGR -> klick “ALTER”
Change “Version” of QMGR to “8.0.0”
Change “CSD” to correct Version (e.g. “FP(8.0.0.2)”)
Klick “update”
Repeat for all migrated QMGRs

Post Migration checks
Run check scripts
root:/> /opt/mw/mqm/bin/mqcheckafteraction.sh

Start sender channels
root:/> for qmgr in ${qmgrnames};do su mqm -c"ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR";done

Check status, compare with check before migration.
root:/> for qmgr in ${qmgrnames};do echo "display chstatus(*) where (STATUS NE RUNNING)" | su mqm - -c ". mqsetenv.sh $qmgr; runmqsc ${qmgr}";done

Check MQ Admin connection
Show the server’s queue managers for copy and paste to MQAdmin
root:/> /opt/mqm800/bin/dspmq
Log in to MQAdmin
Menu: Admin – paste queue manager name
if connection is available, there are values of the queue manager shown
Menu: Site – Agents – paste queue manager name
The agents column “available” has to be “true” 

Stop channels that should not be running (checked before migration) by hand!
root:/> cat /tmp/not-running-channels.txt

Reboot if all qmgrs are under maintenance and repeat step 11 for productive servers
root:/> reboot

Rollback of migrated MQ8 queue manager

Make sure to run all commands in migration scenario as root
xxx:/> sudo su -

stop queue manager
root:/> /opt/mqm800/bin/dspmq -o installation|grep 8\.;echo "enter queue manager to rollback: ";read qmgr; mqstop $qmgr

Save /var/mqm/qmgrs/<qmgr-name> in /var/mq/mqbackup:
root:/> echo "Start Backup of queue manager $qmgr";bfile=MQ8rollback-$qmgr-MQ8-MQ7-`(date +"%Y-%m-%d#%H:%M:%S")`.tar.gz;tar -cvzf /var/mw/mqbackup/${bfile} /var/mqm/qmgrs/$qmgr /var/mqm/log/$qmgr;echo "Backup done, file saved in /var/mw/mqbackup/${bfile}"


Information to user to reset ALL channels informations
root:/> echo ATTENTION: ALL channels need to be resetted manually!

delete queue manager on file system 
root:/> rm -r /var/mqm/qmgrs/${qmgr}/*

unpack newest previously saved backup 
root:/> for file in `ls -1ct /var/mw/mqbackup/MQ8migration-${qmgr}*.tar.gz`; do cd /;tar -xzf $file; cd -;break;done

remove MQ8 installation setting from mqs.ini:
#root:/> ksh "sed -e '/.*DataPath=.*${qmgr}/{n;d}' /var/mqm/mqs.ini"
root:/> ksh "sed -i '/.*DataPath=.*${qmgr}/{n;d}' /var/mqm/mqs.ini"

start QMGR on MQ7
root:/> mqstart $qmgr

Post rollback Test
show queue managers:
root:/> dspmq

Uninstall Tasks

Preparations
Logon as root on BU-Master
xxx:/> sudo su –
Prepare Scripts

root:/> actiondir=/home/mqm/mqserver_7_uninstall
mkdir ${actiondir}
cd ${actiondir}


cat > finalcheck.sh << EOF
#!/bin/ksh


if [[ -x /opt/mqm/800/bin/dspmq ]]
then 
   echo "Error: No MQ 8.0.0 installed"
   rc=1
else
   if [[ 0 -eq \$(/opt/mqm800/bin/dspmq -o installation|grep -c /opt/mqm\)) ]]
   then
      echo "OK: No qmgr assigned to /opt/mqm installation"
      rc=0
   else
      echo "ERROR: Found qmgrs assigned to /opt/mqm installation"
      /opt/mqm800/bin/dspmq -o installation
      rc=1
   fi
fi


exit \$rc


EOF


cat > uninstall7.sh << EOF
#!/bin/ksh


mqstop -all


/opt/mqm800/bin/dspmq


/opt/mw/install/uninstallMQ.sh -version 701 -removerepo -force


if [[ 0 -eq \$(rpm -qa 'MQ*'|grep -v mqm800) ]]
then
   echo "OK: Only MQ 8.0.0 rpms installed"
   rc=0
else
   echo "ERROR: Found non MQ 8.0.0 rpms"
      rpm -qa 'MQ*'|grep -v mqm800
   rc=1
fi
exit \$rc


EOF


chmod 744 *.sh

Prepare server list(s) by netgroup per environment

root:/> echo "mqs_vm_[test|dev|int|prod] select: ";read myenv;group=mqs_vm_${myenv};bung -byNetgroup ${group} > mqservers_${myenv}.list;vi mqservers_${myenv}.list

Final Checks
Logon as root on BU-Master
xxx:/> sudo su –


root:/> actiondir=/home/mqm/mqserver_7_uninstall
mkdir ${actiondir}
cd ${actiondir}


Check for version 7 qmgr

echo "Select list:";ls -1 mqservers_*.list;read list;/root/doforhost/doforhost -w 120 -t 1800 -l ${list} -s ${actiondir}/finalcheck.sh  -T ${actiondir}/${list}_finalcheck.log

	Check returncodes RC from output
grep -e "^# End" -e "^# RC" ${actiondir}/${list}_finalcheck.log

If version 7 qmgr exist – cancel further activities on that server

Check Toolsversion
if not matching a rollout is required (mqserver-update.sh) 
and repeat Check step.
Minimal needed versions:
cgmq-wmq		: 8.0.0.2_6
mqtools-cg-unix	: 8.0.0_14
mw-base		: 1.2.10

	Check with MQAdmin Report: Reviews/Linux Packages

Uninstallation of MQ7

Prepare session
xxx:/>	sudo su -


root:/>	actiondir=/home/mqm/mqserver_7_uninstall;cd ${actiondir}

Uninstall MQServer software for Version 701 

echo "Select list:";ls -1 mqservers_*.list;read list;/root/doforhost/doforhost -w 120 -t 1800 -l ${list} -s ${actiondir}/uninstall7.sh -T ${actiondir}/${list}_uninstall7.log

check logfile and output


grep -e "^# End" -e "^# RC" ${actiondir}/${list}_uninstall7.log

Post Checks

Perform Check

echo "Select list:";ls -1 mqservers_*.list;read list;/root/doforhost/doforhost -w 120 -t 1800 -l ${list} -c /opt/mw/mqm/bin/mqcheckafteraction.sh -T ${actiondir}/${list}_postcheck.log

check logfile and output


grep -e "^# End" -e "^# RC" ${actiondir}/${list}_postcheck.log

Creation of new queue managers
Durchgeführte TESTS Dokument!!
New MQ 7 queue manager

Run the following script to create a new queue manager:

root:/> echo "enter new MQ7 queue manager name to create: ";read qmgr;/opt/mw/mqm/bin/mqcrtqmgr.sh -mqinstpath /opt/mqm -qmgr XBO7011E -logControl ll -logFileSize 16384 -logPrimaryFiles 3 -logSecondaryFiles 508 -logWriteIntegrity DoubleWrite -logBufferPages 1024 -port 1416 -portAdmin 1420 -adminServer mqadmin-int.de050.corpintra.net -adminServerStart QMGR -adminPort 5536 -adminPwd 1234 -monitorServer mqadmin-int.de050.corpintra.net -monitorServerStart QMGR -monitorPort 5557 -monitorPwd 1234 -dataPath /var/mqm/qmgrs -logPath /var/mqm/log -securityModel 1 -logCleanup archive -oldlogFSSize 500 -dataFSSize 1000 -logFSSize 2000 -mqscTemplate iti-ea;
New MQ 8 queue manager

root:/> echo "enter new MQ8 queue manager name to create: ";read qmgr;/opt/mw/mqm/bin/mqcrtqmgr.sh -mqinstpath /opt/mqm800 -qmgr ${qmgr} -logControl ll -logFileSize 16384 -logPrimaryFiles 3 -logSecondaryFiles 508 -logWriteIntegrity DoubleWrite -logBufferPages 1024 -port 1415 -portAdmin 1420 -adminServer mqadmin-int.de050.corpintra.net -adminServerStart QMGR -adminPort 5537 -adminPwd 1234 -monitorServer mqadmin-int.de050.corpintra.net -monitorServerStart QMGR -monitorPort 5557 -monitorPwd 1234 -dataPath /var/mqm/qmgrs -logPath /var/mqm/log -securityModel 1 -logCleanup archive -oldlogFSSize 500 -dataFSSize 1000 -logFSSize 2000 -mqscTemplate iti-ea


Old Version of chapter


root:>/ /opt/mw/install/mqserver-update.sh –updatemq –filelogging /tmp/mqserver-update.log


Linux


Install from local rpms
Install license
root:/> ksh mqlicense.sh -accept -text_only -jre /usr/bin/java

Install MQ
root:/>rpm -ivh MQSeriesRuntime-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesServer-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesSDK-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesClient-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesSamples-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesJava-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesJRE-7.0.1-0.x86_64.rpm
rpm -ivh MQSeriesMan-7.0.1-0.x86_64.rpm
rpm -ivh gsk7bas*4.23*.rpm
rpm -ivh MQSeriesKeyMan-7.0.1-0.x86_64.rpm
Install from central repository
Select RPM repositories server

Linux Installation server (ILSE) 

root:/>	repo=http://linux-repos.edc.corpintra.net/mq-repo

Alternative ITI/EA repo at Sindelfingen

root:/>	repo=http://cgmqrepo.de050.corpintra.net/rpms

Alternative ITI/EA repo at BBAC

root:/>	vi /etc/hosts


# MQ RPM Repo s104slrep785
172.16.20.141 cgmqrepo.bbac.local cgmqrepo


root:/>	repo=http://cgmqrepo/cgmqrepo/rpms


Register repositories

root:/>	zypper ar -f ${repo}/MQ_COMMON CGMQ


root:/>	echo "Env [TEST|DEV|INT|PROD]:"; read env; zypper ar -f ${repo}/MQ70_${env} CGMQ70_${env}


root:/> zypper ref


Fetch installer
root:/> zypper --non-interactive --gpg-auto-import-keys install mqserver-cg-unix

Install MQ Client

root:/> /opt/mw/install/mqserver-install.sh


Deactivate CGMQ repository

root:/>		zypper mr -d CGMQ; zypper lr




Apply latest patch levels as described in ‎1.4.2.3.

Recovery Test

A file with recovery test that must be done before a Server can be go online. \\EMEA.CORPDIR.NET\E050\PUBLIC\ITI-EA\DATEN\STPS\CoC\CoC MQ\Konzepte\MQ_BackupRecovery\recover_test.xls

