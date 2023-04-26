#!/bin/ksh
##############################################################################
#
# Script:         mqstopchannels.sh
#
# Usage:          mqstopchannels.sh <qmgr> <chltype> | <ALL filename>
#
# Description:    select and stop channels
#                     
#
# Remarks:         
#                  
#                  
#                  
#                  
#
# History:         2013-03-02   lreckru     add channel list in file as parameter
#                                           each line must be like "CHANNEL(name)"
#                                           ever exclude SYSTEM.* Objects
#                  2014-11-27   sanzmaj     multiversion
#                               phutzel     checks
#
##############################################################################
#
#set -x
#

# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`

# call the common include
. ${HOME}/mqinclude.sh

if [ $# -lt 2 ] ; then
  echo "Usage : $0 <qmgr> <chltype>"
  echo "No Queue Manager name or channel type supplied"
  echo "You must supply a Queue Manager name and a channel type" 
  echo "ALL | SDR | SVR | RCVR | RQSTR | CLNTCONN | SVRCONN | CLUSSDR | CLUSRCVR  "
  echo "for the channel type ALL you must also supply a filename  (with a list of channel names)"
  echo "each line like 'CHANNEL(XXXXXXX.XXXXXXX.XX)' "
  echo "Use the command dspmq to see the list of installed Queue Managers"
  echo "Exiting ..."
  exit 1
fi

qmgr=$1
CHT=$2
# Checks and set multi-version vars
mqsetenv -m $qmgr

# Check channel list
if [ "${CHT}" = "ALL" ];then
   Obj_Listfile=$3
   extstr="from file $Obj_Listfile"
   if [ ! -f ${Obj_Listfile} ] ; then
      echo "Can't find channel list file ${Obj_Listfile} "
      exit 2
   fi
fi

platform=`uname`

echo "Working with Queue Manager : ${qmgr} and channel type : $CHT $extstr" 
echo "Excluding channels with 'SYSTEM.*' in the name "

if [ "${migrationDir}" = "" ]; then
	if [ $platform = 'CYGWIN_NT-5.1' ]; then
      migrationDir=`dirname $0`/../migration
	else
      migrationDir=/var/mw/mqbackup
	fi
fi

echo "Use migration directory ${migrationDir}"

# ensure we have a migration directory
if [ ! -d $migrationDir ];then
	mkdir $migrationDir
fi

# ensure we have a directory per queue manager
if [ ! -d ${migrationDir}/${qmgr} ];then
	mkdir ${migrationDir}/${qmgr}
fi


if [ -f ${migrationDir}/${qmgr}/"$CHT"_channel.txt ]; then 
   rm ${migrationDir}/${qmgr}/"$CHT"_channel.txt 
fi 
if [ -f ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log ]; then
   echo "" > ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log
else
   touch ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log
fi
HOST=`hostname`
TS=`date +%Y-%m-%d_%H:%M:%S`
echo "$TS - stopping $CHT channels $extstr for Queue Manager: ${qmgr} on host : $HOST" >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log

echo "display chstatus(*) WHERE(STATUS EQ STOPPED)" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr} 2>/dev/null "  > /tmp/${qmgr}_"$CHT"_chs.txt 
retcode=$?
grep AMQ8420 /tmp/${qmgr}_"$CHT"_chs.txt
ret_grep=$? 
# todo parse output and what about other channel types
if [ $retcode != 0 ]; then
   if [ $ret_grep != 0 ]; then 
      echo "Not AMQ8420 : retcode is : $retcode ; exiting"
      exit $retcode 
   fi 
fi	
# list channels to stop
if [ "${CHT}" = "ALL" ];then 
   grep CHLTYPE\(.*\) /tmp/${qmgr}_"$CHT"_chs.txt > ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus_warning.log
else
   grep CHLTYPE\($CHT\) /tmp/${qmgr}_"$CHT"_chs.txt > ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus_warning.log
   cmdstr=`echo "DISPLAY channel(*) WHERE ( chltype EQ $CHT )"`
   echo "${cmdstr}" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}" > /tmp/"${qmgr}"_"$CHT"_dis_channel.txt
   ret_code=$?
   if [ $ret_code != 0 ]; then
      echo " dis channel ret_code is : $ret_code ; exiting "
      exit $ret_code
   fi	 
   grep CHANNEL /tmp/"${qmgr}"_"$CHT"_dis_channel.txt | grep -v SYSTEM. | awk ' { print $1 } ' > ${migrationDir}/${qmgr}/"$CHT"_channel.txt
   Obj_Listfile=${migrationDir}/${qmgr}/"$CHT"_channel.txt
fi
# stop channels 
if [ ! -f "${Obj_Listfile}" ] ; then
   echo "Object list file not readable"
else
   cat ${Obj_Listfile} | while read line
   do

      CHLN=`echo $line | cut -d'(' -f2|cut -d')' -f1 `
      if [ "$CHLN" != "${qmgr}.CL.ADM" ] ;
      then 
      echo "stopping : $line" >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log
      echo "stop $line mode(force)" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}" >   /tmp/"${qmgr}"_"$CHT"_stop_channel.txt
       
      ret_code_sto=$?
      amqerr=`grep AMQ /tmp/"${qmgr}"_"$CHT"_stop_channel.txt`  
      grep -E "AMQ9533|AMQ9531" /tmp/"${qmgr}"_"$CHT"_stop_channel.txt > /dev/null
      ret_grep=$?

      # todo check output AMQ9533 is ok 
      cat /tmp/"${qmgr}"_"$CHT"_stop_channel.txt >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log
      if [ $ret_code_sto != 0 ]; then 
         if [ $ret_grep != 0 ]; then 
            echo "Warning: ${amqerr} for Channel : $line ; continuing"
            # exit $ret_code_sto 
         fi 
      fi	
      else
         echo "skipped Admin SVRCONN channel : $line " >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log
      fi
       
   done
fi

#grep AMQ9533 ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log > ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus_warning.log
TS=`date +%Y-%m-%d_%H:%M:%S`

echo "Waiting for channels to be stopped"
sleep 10

echo "--- SUMMARY ---"
echo "Already stopped channels:"
wc -l ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus_warning.log

echo "$TS - removing work file $CHT_channel.txt" >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_chstatus.log
if [ -f ${migrationDir}/${qmgr}/"$CHT"_channel.txt ] ; then 
   rm ${migrationDir}/${qmgr}/"$CHT"_channel.txt
fi
exit 0
