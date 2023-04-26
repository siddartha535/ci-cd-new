#!/bin/ksh
##############################################################################
#
# Script:         mqstartchannels.sh
#
# Usage:          mqstartchannels.sh <qmgr> <chltype> | <ALL filename>
#
# Description:    start channels
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
#				   2014-11-27	sanzmaj	    multiversion
#								phutzel	    checks
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

if [ "${CHT}" = "ALL" ];then
   Obj_Listfile=$3
   extstr="from file $Obj_Listfile"
   if [ ! -f ${Obj_Listfile} ] ; then
      echo "Can't find channel list file ${Obj_Listfile} "
      exit 2
   fi
fi

echo "Working with Queue Manager : ${qmgr} and channel type : $CHT $extstr" 
echo "Excluding channels with 'SYSTEM.*' in the name "

platform=`uname`

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
if [ ! -d ${migrationDir}/${qmgr} ]; then
   mkdir ${migrationDir}/${qmgr}
fi

if [ -f ${migrationDir}/${qmgr}/"$CHT"_start_channel.txt ]; then 
   rm ${migrationDir}/${qmgr}/"$CHT"_start_channel.txt 
fi 
if [ -f ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log ]; then
   echo "" > ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log
else
   touch ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log
fi
# start selected channels
HOST=`hostname`
TS=`date +%Y-%m-%d_%H:%M:%S`
echo "$TS - starting $CHT channels $extstr for Queue Manager: ${qmgr} on host : $HOST" >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log


if [ "${CHT}" = "ALL" ];then 
	echo "dis channel(*) " | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr} 2>/dev/null  " > /tmp/${qmgr}_"$CHT"_dis_channel_sta.txt
	dis_channel_rc=$?
	if [ $dis_channel_rc != 0 ]; then
		echo " dis channel return code is : $dis_channel_rc ; exiting"
		exit $dis_channel_rc
	fi 
else
	echo "dis channel(*) WHERE ( chltype EQ $CHT )" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}" > /tmp/${qmgr}_"$CHT"_dis_channel_sta.txt
	dis_channel_rc=$?
	if [ $dis_channel_rc != 0 ]; then
		echo " dis channel return code is : $dis_channel_rc ; exiting"
		exit $dis_channel_rc
	fi 
	grep CHANNEL /tmp/${qmgr}_"$CHT"_dis_channel_sta.txt | grep -v SYSTEM. | awk ' { print $1 } ' > ${migrationDir}/${qmgr}/"$CHT"_start_channel.txt
   Obj_Listfile=${migrationDir}/${qmgr}/"$CHT"_start_channel.txt
fi

if [ ! -f "${Obj_Listfile}" ] ; then
   echo "Object list file not readable"
else
   cat ${Obj_Listfile} | while read line
   do

      echo "starting : $line" >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log
      echo "start $line " | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}" > /tmp/"${qmgr}"_"$CHT"_start_channel.txt 
      sta_channel_rc=$?
      amqerr=`grep AMQ /tmp/"${qmgr}"_"$CHT"_start_channel.txt`
      grep -E "AMQ9533|AMQ9531" /tmp/"${qmgr}"_"$CHT"_start_channel.txt > /dev/null
      ret_grep=$?
              
      cat /tmp/"${qmgr}"_"$CHT"_start_channel.txt >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log 

      # todo parse output
      if [ $sta_channel_rc != 0 ]; then
         if [ $ret_grep != 0 ]; then
            echo "Warning: ${amqerr} for Channel : $line ; continuing"
         fi
         #   exit $sta_channel_rc
      fi
      
   done
fi

TS=`date +%Y-%m-%d_%H:%M:%S`
echo "$TS - removing work file $CHT_start_channel.txt" >> ${migrationDir}/${qmgr}/"${qmgr}"_"$CHT"_start_chstatus.log
if [ -f ${migrationDir}/${qmgr}/"$CHT"_start_channel.txt ] ; then 
   rm ${migrationDir}/${qmgr}/"$CHT"_start_channel.txt 
fi 
exit 0
