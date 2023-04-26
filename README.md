
#!/bin/bash

if [[ "$USER" != "root" ]]; then
  echo "Script must be run as  root"
  printf "exiting with code 5 since user is differnt\n"
  exit
fi

SECONDS=0
RED='\033[0;31m'
GREEN='\033[0;32m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color
BLUE_LINE="echo -e "${BLUE}***********************************************${NC}" "
printError   ()  { echo -e   "${RED}ERROR: $@" ${NC}; }
printInfo ()  { echo -e   "${GREEN}INFO: $@" ${NC}; }
getListenerControl() { echo 'dis listener(LISTENER.DEFAULT.01) CONTROL ' | su - mqm -c ". mqsetenv $1 ;runmqsc $1 "| grep CONTROL\( | cut -c 52-57 ; }
getListenerControlqmgr() { echo 'dis listener(LISTENER.DEFAULT.01) CONTROL ' | su - mqm -c ". mqsetenv $1 ;runmqsc $1 "| grep CONTROL\( | cut -c 52-55 ; }
getQmgrImgsched () { echo 'dis qmgr IMGSCHED' | su - mqm -c " . mqsetenv $1; runmqsc $1" | grep AUTO| awk '{print $2}' | cut -c 10-13 ; }
getImgInterval () { echo 'dis qmgr IMGINTVL' | su - mqm -c " . mqsetenv $1 ; runmqsc $1" | grep 1440 | awk  '{print $2}' | cut -c 10-13 ; }
getamqpcontrol() { echo 'dis service(SYSTEM.AMQP.SERVICE) control' | su - mqm -c ". mqsetenv $1; runmqsc $1" | grep MANUAL | awk '{print $2}' | cut -c 9-14 ;}
alterQmgrImgsched () { echo 'alter qmgr IMGSCHED(AUTO)' | su - mqm -c " . mqsetenv $1; runmqsc $1" ; }
alterQmgrImgInvl () { echo 'alter qmgr IMGINTVL(1440)' | su - mqm -c " . mqsetenv $1 ; runmqsc $1" ; }
alterListenerControl () { control=$2; echo "alter listener (LISTENER.DEFAULT.01) TRPTYPE(TCP) "$control" "|su - mqm -c ". mqsetenv $1;runmqsc $1" ; }
disauthinfoidpwldap () { echo 'dis authinfo(SYSTEM.DEFAULT.AUTHINFO.IDPWLDAP) ADOPTCTX' | su - mqm -c " . mqsetenv $1; runmqsc $1" | grep ADOPTCTX | awk '{print $2}'| cut -c 10-12 ; }
disauthinfoidpwos () { echo 'dis authinfo(SYSTEM.DEFAULT.AUTHINFO.IDPWOS) ADOPTCTX' | su - mqm -c " . mqsetenv $1; runmqsc $1" | grep ADOPTCTX | awk '{print $2}'| cut -c 10-12 ; }
migratingQmgr () { su - mqm -c ". mqsetenv $1; /opt/mqm910/bin/setmqm -m $1 -n $(/opt/mqm910/bin/dspmqver -bf 512)" ; }

alterQmgrListenerControlManual () {
    alterListenerControl ${qmgr} "CONTROL(MANUAL)"
    control=$( getListenerControl ${qmgr})
    [ "${control}" = "MANUAL" ]

}

alterQmgrImgschedtoAUTO () { alterQmgrImgsched ${qmgr} ;imgsched=$(getQmgrImgsched ${qmgr} ) ; [ "${imgsched}" = "AUTO" ] ; }
alterQmgrImgIntvlto1440 () { alterQmgrImgInvl ${qmgr} ; imgintvl=$( getImgInterval ${qmgr} ) ; [[ "${imgintvl}" -eq 1440 ]] ; }
alterListenerControltoQMGR () { alterListenerControl ${qmgr} "CONTROL(QMGR)"; control=$( getListenerControlqmgr ${qmgr}); [ "$control" = "QMGR" ] ; }

alterauthinfoLDAPtoYes () {
echo 'alter authinfo(SYSTEM.DEFAULT.AUTHINFO.IDPWLDAP) AUTHTYPE(IDPWLDAP) ADOPTCTX(YES)' | su - mqm -c " . mqsetenv $1; runmqsc $1"
idpwldap=$( disauthinfoidpwldap ${qmgr})
[ $idpwldap = "YES" ]
}

alterauthinfoOStoYes () {
echo 'alt authinfo(SYSTEM.DEFAULT.AUTHINFO.IDPWOS) AUTHTYPE(IDPWOS) ADOPTCTX(YES)' | su - mqm -c " . mqsetenv $1; runmqsc $1"
idpwos=$( disauthinfoidpwos ${qmgr})
[ $idpwos = "YES" ]
}

mqstopQmgr () {
    /opt/mw/mqm/bin/mqstop.sh ${qmgr} &>/dev/null &
    if [[ $(rpm -qa | grep MQSeriesXR | wc -l) -gt 0 ]]; then
        echo "This is an IIB Server, Stopping of ${qmgr} takes minimum of 20 minutes. Please have patience"
        TIMER=2100
    else
        TIMER=20
    fi
    PID=$!;echo $PID;sleep $TIMER
    mqprocess=$(ps -ef |grep -v grep| awk '{print $2}' | grep $PID|  wc -l )
    echo $mqprocess
    if [[ $mqprocess -eq 0 ]]; then
        echo "${qmgr} is stopped"
    else
        echo "${qmgr} is not stopped yet, waiting for more time"
        sleep 20
        mqprocess=$(ps -ef |grep -v grep| awk '{print $2}' | grep $PID|  wc -l )
        if [[ $mqprocess -gt 0 ]]; then
            echo "Exiting the script as the ${qmgr} is not in stopped state. Please stop the ${qmgr} manually and re-run the mq91migration.sh script"
            exit "RC=$RC"
        else
            echo "${qmgr} is stopped successfully. Please proceed with the further steps"
        fi
    fi
    }

mqstartQmgr () {
    /opt/mw/mqm/bin/mqstart.sh ${qmgr}
    ${BLUE_LINE}
    [[ "$(dspmq | grep ${qmgr} | awk '{print $2}' | cut -c 8-14)" = "Running" ]]
}


Date=$(date +%F_%T)
RC=0; success="" ; failure=""
backupDir=/var/mw/mqbackup/migration910
[[ -d ${backupDir} ]] || mkdir ${backupDir}
chown -R mqm:mqm ${backupDir}

qmgrss=$(dspmq |awk '{print $1}' | sed 's/QMNAME//' | sed 's/(//' |sed 's/)//')
if [[ -s /etc/bu/nomqmigration ]]; then
  for qmgr in ${qmgrss}; do
    if [[ $(cat /etc/bu/nomqmigration | grep -w $qmgr | wc -l ) -eq 0 ]]; then
      qmgrs=$(echo $(  echo $qmgr $qmgrs ))
    fi
  done
else
    qmgrs=$(dspmq |awk '{print $1}' | sed 's/QMNAME//' | sed 's/(//' |sed 's/)//')
fi

#totalNosQmgrs=$( echo ${qmgrs} | wc -l }

check_rc ()
{
  returncode=$1; infoString=$2; failString=$3; filecounter=$4; qmgr=$5
  if [[ $returncode -ne 0 ]]; then
    RC=$(( RC + 1 ))
    printError "$failString"
    echo "RC=$RC"
    exit $RC
  else
    if [[ ! -z $infoString ]]; then
      printInfo "$infoString" ; RC=0;
    [ -z "$filecounter" ] ||   touch ${backupDir}/${qmgr}_${filecounter}.txt
    else
      RC=0;
     [ -z $filecounter ] || touch ${backupDir}/${qmgr}_${filecounter}.txt
    fi
  fi
}

chgown ()
{
  local qmgr=$1
  chown mqm:mqm ${qmgr}
  check_rc $? "Ownership changed successfully" "Failed to change the ownweship of ${qmgr}... exiting the program "
}

backup_qmgrs ()
{
  echo -e "${GREEN}Taking Backup of Qmgr ${NC}"
  for qmgr in ${qmgrs}
  do
    echo -e "${GREEN}Step:6 Stopping qmgr/qmgrs ${NC}"
    counter=6
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "${qmgr} is already in stopped state"
    else
        mqstopQmgr ${qmgr}    
        check_rc $? "${qmgr} is stopped successfully" "Exiting the program as the ${qmgr} is not in stopped state" $counter  $qmgr
    fi

    ${BLUE_LINE}

    echo -e "${GREEN}Step:7 Starting Backup of Queue Manager ${qmgr}${NC}"
    counter=7
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "Backup of Qmgr is already taken successfully"
    else
        if [[ -f /etc/bu/mw_noqfilebackup ]]; then
            echo "Excluding Backing of Queue Files for MSB QMGR : $qmgr"
            tar --exclude="/var/mqm/qmgrs/${qmgr}/queues" -cvzf /var/mw/mqbackup/migration910/MQmigration-${qmgr}-MQv9-to-MQv91.tar.gz /var/mqm/qmgrs/${qmgr} /var/mqm/log/${qmgr}
        else
            tar -cvzf /var/mw/mqbackup/migration910/MQmigration-${qmgr}-MQv9-to-MQv91.tar.gz /var/mqm/qmgrs/${qmgr} /var/mqm/log/${qmgr}
        fi
        [[ -f ${backupDir}/MQmigration-${qmgr}-MQv9-to-MQv91.tar.gz ]]
        check_rc $? "Backup has been successfully, backup can be founder under /var/mw/mqbackup/migration910/${bfile}" "We are exiting the program as the backup file is not found under the backup dir" "${counter}" "${qmgr}"
    fi
  done
}

channels_queues_status_check_beforemigration ()
{
  local qmgr=$1
  echo -e "${GREEN}Start the channel and check the channel & queue status for ${qmgr} ${NC}"
  . mqsetenv ${qmgr}
  su mqm -c "ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR"
  sleep 10;
  echo "display chstatus(*) where (STATUS NE RUNNING)" |su mqm -c "runmqsc ${qmgr}" > /var/mw/mqbackup/migration910/${qmgr}_${Date}_retrying_chstatus_beforemigrtion.log
  echo "display qs(*) where (CURDEPTH GT 0)" |su mqm -c "runmqsc ${qmgr}" > /var/mw/mqbackup/migration910/${qmgr}_${Date}_qstatus_beforemigration.log  
}

channels_queues_status_check_aftermigration ()
{
  local qmgr=$1
  echo -e "${GREEN}Start the channel and check the channel & queue status for ${qmgr} ${NC}"
  . mqsetenv ${qmgr}
  su mqm -c "ksh /opt/mw/mqm/bin/mqstartchannels.sh ${qmgr} SDR"
  sleep 10;
  echo "display chstatus(*) where (STATUS NE RUNNING)" |su mqm -c "runmqsc ${qmgr}" > /var/mw/mqbackup/migration910/${qmgr}_${Date}_retrying_chstatus_aftermigration.log
  echo "display qs(*) where (CURDEPTH GT 0)" |su mqm -c "runmqsc ${qmgr}" > /var/mw/mqbackup/migration910/${qmgr}_${Date}_qstatus_aftermigration.log
}


Update_config ()
{
  file=$1
  string=$2
  while read line
  do
    if [[ $( echo "$line" | grep -v "^#"|wc -l ) -eq 1 ]]; then
      if [ -n "$line" ]; then
      last_line=$(echo $line)
      fi
    fi
  done<$file
 
  sed -i "/${last_line}/a ${string}"  ${file}
  if [[ $(cat ${file}| grep ${string} | wc -l ) -eq 1 ]]; then
    echo " $file is updated successfully ..."
  fi
}

migrating_qmgrs ()
{
  echo -e  "${GREEN}Starting the migration process${NC}"
  for qmgr in ${qmgrs}
  do
    chgown "/etc/mw/mqm/locals_${qmgr}"
  done
  chgown "/var/mqm/errors"
  chgown "/var/mw/backup"
  chgown "/var/mqm/trace"
 
  ${BLUE_LINE}
   
  echo -e "${GREEN}Step:1 Taking the backup of the Queue Manager  status${NC}"
  for qmgr in ${qmgrs}; do
  counter=1
  if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "Backup of /var/mw/backup/current/mqm/${qmgr} is already taken"
  else
        /opt/mw/bin/mw_backup.sh
        check_rc $? "Backup completed successfuly" "Program is getting exit due to failure of backup process" "${counter}" "${qmgr}"
  fi
  done

  ${BLUE_LINE}
 
  echo -e "${GREEN}Step:2 Performing clean-up of transactions logs${NC}"
  for qmgr in ${qmgrs}; do
    counter=2
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "MQ Logcleanup is already completed successfully"
    else
        /bin/su - mqm -c "/opt/mw/mqm/bin/mqlogcleanup.sh ${qmgr}"
        check_rc $? "Logcleanup happened successfully for ${qmgr}" "We are exiting the program due to failure of logcleanup on ${qmgr}" "${counter}" "${qmgr}"
    fi
  done

  ${BLUE_LINE}
 
  echo -e "${GREEN}Step:3 channel & queue status check${NC}"
  for qmgr in ${qmgrs}
  do
    counter=3
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "Channel and Queue status checks were already done successfully for ${qmgr} and the output can be found under ${backupDir}"
    else
        channels_queues_status_check_beforemigration ${qmgr}
        check_rc $? "Channel and Queue status check files for ${qmgr} are available under ${backupDir}" "Exiting the program as the Channel and the Queue Stauts files are not found for ${qmgr}" "${counter}" "${qmgr}"
        echo "$(ls -ltr ${backupDir}/*beforemigration*)"
    fi
 
        ${BLUE_LINE}

    echo -e "${GREEN}Step:4 Stopping channels${NC}"
    counter=4
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "Channels are already stopped on ${qmgr}"
    else
        /opt/mw/mqm/bin/mqstopchannels.sh ${qmgr} ALL
        check_rc $? "Channels are stopped successfully on ${qmgr}.." "Exiting the program as the CHANNELS are NOT stopped on ${qmgr}.." "${counter}" "${qmgr}"
    fi

        ${BLUE_LINE}

    echo -e "${GREEN}Step:5 Changing the Qmgr LISTENER to MANUAL${NC}"
    counter=5
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "Qmgr LISTENER is already changed to MANUAL"
    else
        alterQmgrListenerControlManual ${qmgr}
        check_rc $? "LISTENER for the ${qmgr} is set to MANUAL.." "Exiting the program as the LISTENER is not set to MANUAL on ${qmgr}.." "${counter}" "${qmgr}"
    fi
  done

        ${BLUE_LINE}

   backup_qmgrs # Backing up Qmgrs
 
        ${BLUE_LINE}

   echo -e "${GREEN}Waiting for 5 seconds before proceeding with the migration${NC}"
   sleep 5
 
        ${BLUE_LINE}

    echo -e "${GREEN}Step:8 Migrating Qmgrs on server $(hostname) from MQv9 to MQ9.1${NC}"
    for qmgr in $qmgrs
    do
        counter=8
        if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
            echo "${qmgr} is already migrated to MQv9.1"
        else
            migratingQmgr ${qmgr}    
            check_rc $? "Migrated the ${qmgr} successfully to MQv9.1" "Exiting the script as migration is not successful for ${qmgr}, please check" "${counter}" "${qmgr}"
        fi
    done
 
        ${BLUE_LINE}
 
    echo -e "${GREEN}Verifing the ${qmgr} installation${NC}"
    echo "$(dspmq -o installation)"
       
        ${BLUE_LINE}

    echo -e "${GREEN} Make the MQv9.1 InstallationName as Primary Installation${NC}"
    echo -e "${GREEN}Step:9 Unset the Primary installation from /opt/mqm900${NC}"
    for qmgr in ${qmgrs};
    do
        counter=9
        if [[ -f ${backupDir}/${qmgr}_9.txt ]]; then
            echo "MQv9 is already unset as Primary Installation on ${qmgr}"
        else
            echo "$(/opt/mqm900/bin/setmqinst -x -p /opt/mqm900)"
            check_rc $? "Proceed with setting MQv9.1 as Primary Installation on ${qmgr}" "Program is exiting due to the failure of unsetting the MQv9 as Primary installation on ${qmgr}... Please check errors and re-run the script again" "${counter}" "${qmgr}"
        fi
   
    echo -e "${GREEN}Step:10 Set the Primary installation to /opt/mqm910${NC}"
    counter=10
    if [[ -f ${backupDir}/${qmgr}_10.txt ]]; then
        echo "MQv9.1 is already set as Primary Installation on ${qmgr}"
    else
        echo "$(/opt/mqm900/bin/setmqinst -i -p /opt/mqm910)"
        check_rc $? "MQ9.1 is set as Primary Installation on ${qmgr}" "Program is getting exit due to failure of setting MQv9.1 as Primary Installation on ${qmgr}. Please check errors and re run the script again" "${counter}" "${qmgr}"
    fi
   
    done

    InstVer=$(dspmqver -i | grep -e "Version\|Primary")
    echo "$InstVer"
 
        ${BLUE_LINE}
 
    echo -e "${GREEN}Adding MSGSEVERITY & JSON properties to the qmgr locals file ${NC}"
    for qmgr in ${qmgrs}
    do
        echo -e "${GREEN}Step:11 Adding MSGSEVERITY property to Qmgr locals file ${NC}"
        counter=11
        if [[ -f ${backupDir}/${qmgr}_11.txt ]]; then
            echo "MSGSEVERITY property is already added in the  locals file of ${qmgr}"
        else
            Update_config  "/etc/mw/mqm/locals_${qmgr}" "MSGSEVERITY=NO"
            [[ "$(cat /etc/mw/mqm/locals_${qmgr}| grep MSGSEVERITY | wc -l)" -eq 1 ]]
            check_rc $? "MSGSEVERITY property is added to the locals_${qmgr} file" "Program is getting exit due to failure of adding MSGSEVERITY properties on ${qmgr}... Please check errors and re-run the script again" "${counter}" "${qmgr}"
        fi
   
    echo -e "${GREEN}Step:12 Adding JSON property to Qmgr locals file ${NC}"
    counter=12
    if [[ -f ${backupDir}/${qmgr}_12.txt ]]; then
        echo "JSON property is already added in the locals file of ${qmgr}"
    else
        Update_config  "/etc/mw/mqm/locals_${qmgr}" "JSON=YES"
        [[ "$(grep JSON /etc/mw/mqm/locals_${qmgr}| wc -l)" -eq 1 ]]
        check_rc $? "JSON property is added to the locals_${qmgr} file" "Program is getting exit due to failure of adding JSON properties on ${qmgr}... Please check errors and re run  the script again" "${counter}" "${qmgr}"
    fi
    done
 
        ${BLUE_LINE}

    echo -e "${GREEN}Step:13 Adding LogManagement entry in qm.ini ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=13
        if [[ "$(grep 'LogManagement=' /var/mqm/qmgrs/${qmgr}/qm.ini |wc -l)" -eq 1 ]]; then
            echo "Already LogManagement Entry is added to the qm.ini file of ${qmgr}"
        else
            logcleanup=$(grep "LOGCLEANUP=archive" /etc/mw/mqm/locals_${qmgr} | wc -l)
            if [[ "$(grep "LOGCLEANUP=archive" /etc/mw/mqm/locals_${qmgr} | wc -l)" -eq 1 ]]; then
                sed -i '/LogWriteIntegrity=.*/a LogManagement=Archive' /var/mqm/qmgrs/${qmgr}/qm.ini
                sed -i 's/LogManagement=Archive/   LogManagement=Archive/g' /var/mqm/qmgrs/${qmgr}/qm.ini
                logmanagement=$(grep 'LogManagement=Archive' /var/mqm/qmgrs/${qmgr}/qm.ini| wc -l)
                [[ "$logmanagement" -eq 1 ]]
                check_rc $? "LogManagement is set as Archive in qm.ini file of ${qmgr}" "LogManagement is not set as Archive in ${qmgr} qm.ini file of ${qmgr}" "${counter}" "${qmgr}"
            else
                sed -i '/LogWriteIntegrity=.*/a LogManagement=Automatic' /var/mqm/qmgrs/${qmgr}/qm.ini
                sed -i 's/LogManagement=Automatic/   LogManagement=Automatic/g' /var/mqm/qmgrs/${qmgr}/qm.ini
                logmanagement=$(grep 'LogManagement=Automatic' /var/mqm/qmgrs/${qmgr}/qm.ini| wc -l)
                [[ "$logmanagement" -eq 1 ]]
                check_rc $? "LogManagement is set as Automatic in qm.ini file of ${qmgr}" "LogManagement is not set as Automatic in ${qmgr} qm.ini file of ${qmgr}" "${counter}" "${qmgr}"
            fi
        fi
    done

        ${BLUE_LINE}

    echo -e "${GREEN}Step:14 Adding ChlauthEarlyAdopt stanza in qm.ini ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=14
        if [[ "$(grep 'ChlauthEarlyAdopt=' /var/mqm/qmgrs/${qmgr}/qm.ini |wc -l)" -eq 1 ]]; then
            echo "Already ChlauthEarlyAdopt stanza is added to the qm.ini file of ${qmgr}"
        else
            sed -i '/AdoptNewMCACheck=.*/a ChlauthEarlyAdopt=Y' /var/mqm/qmgrs/${qmgr}/qm.ini
            sed -i 's/ChlauthEarlyAdopt=Y/   ChlauthEarlyAdopt=Y/g' /var/mqm/qmgrs/${qmgr}/qm.ini
            chlauthearlyadopt=$(grep 'ChlauthEarlyAdopt=Y' /var/mqm/qmgrs/${qmgr}/qm.ini| wc -l)
            [[ "${chlauthearlyadopt}" -eq 1 ]]
            check_rc $? "ChlauthEarlyAdopt stanza is added to qm.ini file of ${qmgr}" "ChlauthEarlyAdopt stanza is not added to qm.ini file of ${qmgr}" "${counter}" "${qmgr}"
        fi
    done

        ${BLUE_LINE}

    echo -e "${GREEN}Step:15 Starting all the Qmgrs ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=15
        if [[ -f ${backupDir}/${qmgr}_15.txt ]]; then
            echo "This action is already taken care"
        else
            mqstartQmgr ${qmgr}
            check_rc $? "${qmgr} is in Running state... Please proceed with the furhter steps" "{$qmgr} is not in running state, reatart the qmgrs again" "${counter}" "${qmgr}"
        fi
    done
 

        ${BLUE_LINE}
 
    echo -e "${GREEN} Change Qmgr property to schedule the media image backup automatic once in 24 hours ${NC}"
    for qmgr in ${qmgrs}
    do
        echo -e "${GREEN}Step:16 Altering IMGSCHED to AUTO ${NC}"
        counter=16
        if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
            echo "IMGSCHED of the ${qmgr} is already to set to AUTO"
        else
            alterQmgrImgschedtoAUTO ${qmgr}
            check_rc $? "IMGSCHED is set to AUTO on ${qmgr}... proceed with changing the IMGINTVL" "Exiting the program as IMGSCHED is not set to AUTO on ${qmgr}.. Please verify" "${counter}" "${qmgr}"
        fi

        ${BLUE_LINE}
   
    echo -e "${GREEN}Step:17 Altering IMGINTVL to 24 Hrs ${NC}"
    counter=17
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "IMGINTVL of the ${qmgr} is already set to 24 hours"
    else
        alterQmgrImgIntvlto1440 ${qmgr}
        check_rc $? "IMGINTVL is set to 24 hours on ${qmgr}. Proceed further" "Exiting the program as IMGINTVL is not set to 24 hours on ${qmgr}. Please verify" "${counter}" "${qmgr}"
    fi
       
    echo -e "${GREEN}Step:18 Altering ADOPTCTX value of IDPWLDAP to YES ${NC}"
    counter=18
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "ADOPTCTX value of IDPWLDAP is already set to YES"
    else
        alterauthinfoLDAPtoYes ${qmgr}
        check_rc $? "ADOPTCTX value for SYSTEM.DEFAULT.AUTHINFO.IDPWLDAP is set to YES" "ADOPTCTX value for SYSTEM.DEFAULT.AUTHINFO.IDPWLDAP is not set to YES, please re-run the script" "${counter}" "${qmgr}"

    fi    
        ${BLUE_LINE}
    echo -e "${GREEN}Step:19 Altering ADOPTCTX value of IDPWOS to YESY ${NC}"
    counter=19
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "ADOPTCTX value of IDPWOS is already set to YES"
    else
        alterauthinfoOStoYes ${qmgr}
        check_rc $? "ADOPTCTX value for SYSTEM.DEFAULT.AUTHINFO.IDPWOS is set to YES" "ADOPTCTX value for SYSTEM.DEFAULT.AUTHINFO.IDPWOS is not set to YES, please re-run the script" "${counter}" "${qmgr}"

    fi
    done
 
        ${BLUE_LINE}
 
    echo -e "${GREEN}Step:20 Change the Control of the LISTENER to QMGR ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=20
        if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
            echo "${qmgr} LISTENER Control is already set to QMGR"
        else
            alterListenerControltoQMGR ${qmgr}
            check_rc $? "LISTENER is set to QMGR on ${qmgr}." "Exiting program as the LISTENER is not set to QMGR on ${qmgr},if this operation fails for the second time, then go with the ROllback decision" "${counter}" "${qmgr}"
        ${BLUE_LINE}
        fi
    done
 
        ${BLUE_LINE}
 
    echo -e "${GREEN}Step:21 CONTROL of SYSTEM.AMQP.SERVICE should be MANUAL ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=21
        if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
            echo "CONTROL of AMQP service is already checked in ${qmgr}"
        else
            echo  'dis service(SYSTEM.AMQP.SERVICE) control' | su - mqm -c ". mqsetenv ${qmgr} ; runmqsc ${qmgr}"
        ${BLUE_LINE}
            check_rc $? "CONTROL of AMQP service is set to MANUAL on ${qmgr}" "CONTROL of AMQP service not yet set to MANUAL on ${qmgr}" "${counter}" "${qmgr}"
        fi
    done
 
        ${BLUE_LINE}
 
    echo -e "${GREEN}Step:22 Verify JSON files generated under /var/mqm/errors & Qmgr error logs ${NC}"
    [[ $( ls  /var/mqm/errors/*json | wc -l) -eq 3 ]]
    check_rc $? "3 JSON files are available under /var/mqm/errors" "3 JSON files are not available under /var/mqm/errors"
 
 
    for qmgr in ${qmgrs}
    do
        [[ $(ls  /var/mqm/qmgrs/${qmgr}/errors/*.json | wc -l) -eq 3 ]]
        check_rc $? "3 JSON files are available under /var/mqm/qmgrs/${qmgr}/errors/*.json on ${qmgr}" "3 JSON files are not available under /var/mqm/qmgrs/${qmgr}/errors/*.json on ${qmgr}"
    done
       
        ${BLUE_LINE}
 
    echo -e "${GREEN}Restart the Queue Manager on $(hostname) ${NC}"
    echo -e "${GREEN}Step:23 Stopping the qmgr/qmgrs ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=23
        if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
            echo "${qmgr} is already in stopped state"
        else
            mqstopQmgr ${qmgr}
            check_rc $? "${qmgr} is in stopped state" "Exiting the program as the ${qmgr} is not in stopped state" "${counter}" "${qmgr}"
        fi

    echo -e "${GREEN}Step:24 Starting the qmgr/qmgrs ${NC}"
    counter=24
    if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
        echo "this action is already taken"
    else
        mqstartQmgr ${qmgr}
        check_rc $? "${qmgr} is in running state... Proceeding with the next step" "Exiting the program as the ${qmgr} is not in Running state" "${counter}" "${qmgr}"
    fi
   
        ${BLUE_LINE}
   
    done
 
        ${BLUE_LINE}
 
    echo -e "${GREEN}Step:25 Performing Channel & Queue Status Checks ${NC}"
    for qmgr in ${qmgrs}
    do
        counter=25
        if [[ -f ${backupDir}/${qmgr}_${counter}.txt ]]; then
            echo "Channel and Queue Status checks are already performed"
        else
            channels_queues_status_check_aftermigration ${qmgr}  
            check_rc $? "Channel & Queue status files for ${qmgr} is available under ${backupDir}" "Exiting the program as the Channel & Queue Staus Check files for ${qmgr} is not available.. Please check" "${counter}" "${qmgr}"
        echo "$(ls -ltr ${backupDir}/*aftermigration* )"
        diff /var/mw/mqbackup/migration910/${qmgr}_*_retrying_chstatus_beforemigrtion.log /var/mw/mqbackup/migration910/${qmgr}_*_retrying_chstatus_aftermigration.log
        echo "if you notice any channels other than in Running state after the migration, please investigate"
        fi
    done

        ${BLUE_LINE}

    echo -e "${GREEN}Step:26 Stopping the qmgr/qmgrs ${NC}"
    for qmgr in ${qmgrs}
    do
        /opt/mw/mqm/bin/mqstop.sh -all
        sleep 10
      #  [[ "$(ps -ef | grep ${qmgr} |grep -v grep | wc -l)" -eq 0 ]]
        check_rc $? "Qmgr/qmgrs are in stopped state.Proceed in rebooting the server." "Exiting the program as the qmgrs are not in stopped state" "${counter}" "${qmgr}"
    done

}

echo "Time taken for MQv9.1 Migration is $SECONDS"
# End of migration function

echo -e "${GREEN}Check  MQ9.1 is installed ${NC}"
mqversion=$(dspmqver -i | grep '9.1.' | grep -i Version | awk -F':' '{printf $2}')
if [ ! -z $mqversion ]; then
    migrating_qmgrs
else
    RC=$(( RC + $? ))
    printError "Exiting the program as the MQv9.1 is not installed... Please execute the script mq91install.sh in bu-master to install MQv9.1 "
    echo RC=${RC}
    exit $RC
fi
