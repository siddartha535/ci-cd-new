#!/bin/ksh
##############################################################################
#
# Script:         mqbackup.sh
#
# Usage:          mqbackup.sh <backupdir>
#
# Description:    This script does a complete backup of a qmgr with 
#						configuration files 
#                  
#                     
#
# Remarks:         
#                  
#                  
# Todos:           Return code checking                 
#                  
#
# History:         2011-10-24   fdressle     initial version
#                  2013-13-21   lreckru      add save size of oldlogfiles FS              
#                  2014-11-17   sanzmaj      multiversion
#                               phutzel      checks ... no modifications needed
#                  2015-03-19   lreckru      change saveqmgr to dmpmqcfg for Version > 701  
#                  2015-04-28   lreckru      add: exit $rcg - return a value for errorhandling
##############################################################################

# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`
PRG=$0
# call the common include
. ${HOME}/mqinclude.sh

MQ_BACKUP_DIR=$1
export LVM_SUPPRESS_FD_WARNINGS=1

# change owner to mqm
chown mqm:mqm ${MQ_BACKUP_DIR}

# general mqs.ini
echo "Copy /var/mqm/mqs.ini to '${MQ_BACKUP_DIR}' ..."
cp -p /var/mqm/mqs.ini ${MQ_BACKUP_DIR}

# Block IP configuration
#echo "Copy '${MQTOOLS_ETC}/*.bip' to '${MQ_BACKUP_DIR}' ..."
#cp -p ${MQTOOLS_ETC}/*.bip ${MQ_BACKUP_DIR}

# lvm sizing
linuxversion=$(grep VERSION /etc/S*SE-brand |awk '{print $3}')
if [[ ${linuxversion} -eq 11 ]]; then
    sectors=`lvdisplay -c ${DEVICE_DIR}/data1vg/lvvar_mw_mqbackup|cut -d: -f7`
fi
if [[ ${linuxversion} -eq 12 ]]; then
        qvalue=$(lvdisplay -c ${DEVICE_DIR}/backupvg/lvvar_mw_qbackup 2>&1)
        if [[ $? -eq 0 ]]; then
                sectors=$(echo ${qvalue} | awk -F ":" '{print $7}')
        fi
        mqvalue=$(lvdisplay -c ${DEVICE_DIR}/backupvg/lvvar_mw_mqbackup 2>&1)
        if [[ $? -eq 0 ]]; then
                sectors=$(echo ${mqvalue} | awk -F ":" '{print $7}')
        fi
#           sectors=`lvdisplay -c ${DEVICE_DIR}/backupvg/lvvar_mw_mqbackup|cut -d: -f7`
fi
BACKUP_SIZE=`expr $sectors / 2048`
    
echo "Copy lvm.ini to '${MQ_BACKUP_DIR}' ..."   
cat > ${MQ_BACKUP_DIR}/lvm.ini << EOF
BACKUP_SIZE=$BACKUP_SIZE
EOF

#adding the vgs, pvs, lsscsi to a new file diskdetails.txt
echo "Storing the disk information in diskdetails.txt on /var/mw/backup/current/mqm"
diskfile="diskdetails.txt"

echo "Output of lsscsi:" >> ${MQ_BACKUP_DIR}/${diskfile}
echo "=================" >> ${MQ_BACKUP_DIR}/${diskfile}
lsscsi >> ${MQ_BACKUP_DIR}/${diskfile}

echo "Output of vgs:" >> ${MQ_BACKUP_DIR}/${diskfile}
echo "==============" >> ${MQ_BACKUP_DIR}/${diskfile}
vgs >> ${MQ_BACKUP_DIR}/${diskfile}

echo "Output of pvs:" >> ${MQ_BACKUP_DIR}/${diskfile}
echo "==============" >> ${MQ_BACKUP_DIR}/${diskfile}
pvs >> ${MQ_BACKUP_DIR}/${diskfile}


#new
rcg=0

# For each configured queue manager
ls -1 ${MQTOOLS_ETC}/locals_* | while read LOCALSFILE
do
   
   
   # just filename
   FILENAME=`basename ${LOCALSFILE}`

   # extract name of qmgr
   qmgr=${FILENAME#locals_}
   QMGR_BACKUP_DIR=${MQ_BACKUP_DIR}/${qmgr}
   [[ -d ${QMGR_BACKUP_DIR} ]] || mkdir ${QMGR_BACKUP_DIR}
   chown mqm:mqm ${QMGR_BACKUP_DIR}
   
   mqsetenv -m ${qmgr}
   
   # save locals_${qmgr} file
   echo "Copy '${LOCALSFILE}' to '${MQ_BACKUP_DIR}/${qmgr}' ..."
   cp -p ${LOCALSFILE} ${MQ_BACKUP_DIR}/${qmgr}

   # save dlq rule
   echo "Copy '${MQTOOLS_ETC}/runmqdlq_${qmgr}_rules.conf' to '${MQ_BACKUP_DIR}/${qmgr}' ..."
   cp -p ${MQTOOLS_ETC}/runmqdlq_${qmgr}_rules.conf ${MQ_BACKUP_DIR}/${qmgr}
 
   echo "Copy '${qmgr}.mqsc' and '${qmgr}.aut' to '${QMGR_BACKUP_DIR}' ..."
   if [ $mqversion -ge 710 ]; then
         ### 7.1 new command :  dmpmqcfg  
         ${su_cmd_pre} ". ${MQTOOLS_HOME}/bin/mqsetenv.sh $qmgr;${MQ_HOME}/bin/dmpmqcfg -m ${qmgr} -a" > ${QMGR_BACKUP_DIR}/${qmgr}.mqsc 
         rc=$?
         ((rcg=$rcg+$rc))    
         [[ $rc -eq 0 ]] && ${su_cmd_pre} ". ${MQTOOLS_HOME}/bin/mqsetenv.sh $qmgr;${MQ_HOME}/bin/dmpmqcfg -m ${qmgr} -o setmqaut" >  ${QMGR_BACKUP_DIR}/${qmgr}.aut        
         rc=$?
      else 
         # call saveqmgr for config and OAM      
         ${su_cmd_pre} "${MQTOOLS_HOME}/bin/saveqmgr -m ${qmgr} -f ${QMGR_BACKUP_DIR}/${qmgr}.mqsc -z ${QMGR_BACKUP_DIR}/${qmgr}.aut"
         rc=$?
      fi
 
    if [[ ! -s ${QMGR_BACKUP_DIR}/${qmgr}.mqsc ]];then
    	p_error "${QMGR_BACKUP_DIR}/${qmgr}.mqsc is empty"
    	((rcg=$rcg+1))
    fi
    if [[ ! -s ${QMGR_BACKUP_DIR}/${qmgr}.aut ]];then
    	p_error "${QMGR_BACKUP_DIR}/${qmgr}.aut is empty"
    	((rcg=$rcg+1))
    fi
 
    #new
    set -o pipefail   
 
   # lvm sizing
   #sectors=`lvdisplay -c ${DEVICE_DIR}/data1vg/${qmgr}_lv|cut -d: -f7`
   #DATA_SIZE=`expr $sectors / 2048`
   
   #sectors=`lvdisplay -c ${DEVICE_DIR}/data2vg/${qmgr}log_lv|cut -d: -f7`
   #LOG_SIZE=`expr $sectors / 2048`

    #new
   sectors=`lvdisplay -c ${DEVICE_DIR}/data1vg/${qmgr}_lv|cut -d: -f7`
    ret=$?
    if [[ $ret -ne 0 ]]; then
        ((rcg=$rcg+1))
    fi
   DATA_SIZE=`expr $sectors / 2048`
    
   sectors=`lvdisplay -c ${DEVICE_DIR}/data2vg/${qmgr}log_lv|cut -d: -f7`
    ret=$?
    if [[ $ret -ne 0 ]]; then
        ((rcg=$rcg+1))
    fi
   LOG_SIZE=`expr $sectors / 2048`

   
   #sectors=`lvdisplay -c ${DEVICE_DIR}/data2vg/${qmgr}oldlog_lv|cut -d: -f7`
   #OLDLOG_SIZE=`expr $sectors / 2048`
   
    #new
    LOGCLNUP=$(cat ${QMGR_BACKUP_DIR}/locals_${qmgr} | grep -i LOGCLEANUP | awk -F "=" '{print $2}')
    if [[ ${LOGCLNUP} != "purge" ]]; then
   sectors=`lvdisplay -c ${DEVICE_DIR}/data2vg/${qmgr}oldlog_lv|cut -d: -f7`
        rc=$?
        if [[ $rc -eq 5 ]]; then
                echo "${DEVICE_DIR}/data2vg/${qmgr}oldlog_lv not found"
        fi
        if [[ $rc -eq 0 ]]; then
   OLDLOG_SIZE=`expr $sectors / 2048`
        fi
    else
        OLDLOG_SIZE=""
    fi
   

   echo "Copy lvm.ini to '${QMGR_BACKUP_DIR}' ..."   

   cat > ${QMGR_BACKUP_DIR}/lvm.ini << EOF
DATA_SIZE=$DATA_SIZE
LOG_SIZE=$LOG_SIZE
OLDLOG_SIZE=$OLDLOG_SIZE
EOF

   # qm.ini
   echo "Copy qm.ini to '${QMGR_BACKUP_DIR}' ..."
   cp -p /var/mqm/qmgrs/${qmgr}/qm.ini ${QMGR_BACKUP_DIR}
   
   #QMQMOBJCAT
   echo "Copy QMQMOBJCAT to '${QMGR_BACKUP_DIR}' ..."
   cp -p /var/mqm/qmgrs/${qmgr}/qmanager/* ${QMGR_BACKUP_DIR}

   # ssl
   echo "Copy ssl to '${QMGR_BACKUP_DIR}' ..."
   cp -pr /var/mqm/qmgrs/${qmgr}/ssl ${QMGR_BACKUP_DIR}
   chown mqm:mqm ${QMGR_BACKUP_DIR}/*
   ((rcg=$rcg+$rc))
done
exit $rcg
