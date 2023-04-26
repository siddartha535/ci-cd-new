#!/bin/ksh
##############################################################################
#
# Script:         mqdisablequeues.sh
#
# Usage:          mqdisablequeues.sh <qmgr> [filename]
#
# Description:    disable queues for put
#                     
#
# Remarks:         
#                  
#                  
#                  
#                  
#
# History:         2013-03-02   lreckru     add obj list in file as parameter
#                                           each line must be like "QUEUE(name)"
#                  2014-11-27   sanzmaj     multiversion
#                               phutzel     checks
#                                           
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


if [ $# -lt 1 ] ; then
  echo "Usage : $0 <qmgr> [filename]"
  echo "No Queue Manager name supplied"
  echo "You must supply a Queue Manager name "
  echo "Use the command dspmq to see the list of installed Queue Managers"
  echo "Exiting ..."
  echo "for using a object list in a file, each line must look like  'QUEUE(name of queue)'"
  exit 1
fi
qmgr=$1

# Checks and set multiple-version vars
mqsetenv -m $qmgr

FileParam=false
if [ $# -eq 2 ];then
    Obj_Listfile=$2
    FileParam=true
    extstr="and objects from file $Obj_Listfile"
  if [ ! -f ${Obj_Listfile} ] ; then
      echo "Can't find object list file ${Obj_Listfile} "
      exit 2
  fi
fi

echo "Working with Queue Manager : ${qmgr}  $extstr"

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

if [ -f ${migrationDir}/${qmgr}/queue.txt ]; then 
  rm ${migrationDir}/${qmgr}/queue.txt 
fi 
if [ -f ${migrationDir}/${qmgr}/queue_status.log ];
then
  echo "" > ${migrationDir}/${qmgr}/queue_status.log
else
  touch ${migrationDir}/${qmgr}/queue_status.log
fi
# disable put for qmgr
HOST=`hostname`
TS=`date +%Y-%m-%d_%H:%M:%S`
echo "$TS - disabling queue PUT for Queue Manager: ${qmgr} on host : $HOST" >> ${migrationDir}/${qmgr}/queue_status.log
echo "searching for already put disabled queues...."
echo "display queue(*) where ( PUT EQ DISABLED ) " | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}"  > /tmp/${qmgr}_disable_qlocal.txt
ret_disable=$?
grep AMQ8147 /tmp/${qmgr}_disable_qlocal.txt
ret_grep=$? 
# todo parese output; not a error AMQ8147: WebSphere MQ object * not found. 
if [ $ret_disable != 0 ];then
  if [ $ret_grep != 0 ]; then 
            echo "Not AMQ8147 : ret_disable is : $ret_disable ; exiting"
            exit $ret_disable
  fi
      
fi	
grep QUEUE  /tmp/${qmgr}_disable_qlocal.txt | awk ' { print $1 " " $2} ' > ${migrationDir}/${qmgr}/queue.already.disabled.log
echo "searching for queues to put disable..."


if [ $FileParam = false ];
then
	echo "display queue(*)" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}"  > /tmp/${qmgr}_dis_ql_for_dis.txt
	ret_dis_for_dis=$?
	if [ $ret_dis_for_dis != 0 ]; then
		echo "ret_dis_for_dis is : $ret_dis_for_dis ; exiting "
		exit $ret_dis_for_dis
	fi	
else
   echo "" > /tmp/${qmgr}_dis_ql_for_dis.txt
   while read line
   do
	  	echo "display $line TYPE" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}" | grep -v "display $line" >> /tmp/${qmgr}_dis_ql_for_dis.txt
		ret_dis_ena=$?
		if [ $ret_dis_ena != 0 ]; then
			echo "ret_dis_ena is : $ret_dis_ena ; exiting "
			exit $ret_dis_ena
		fi	   
   done < ${Obj_Listfile}
fi
    awk '{ if(match($1,"QUEUE\\(") > 0) { queue=$1 } if(match($1,"TYPE\\(") > 0) { print queue " " $1} if(match($2,"TYPE\\(") > 0) {print queue " " $2} }' /tmp/${qmgr}_dis_ql_for_dis.txt > ${migrationDir}/${qmgr}/queue.txt
    Obj_Listfile=${migrationDir}/${qmgr}/queue.txt


echo "disable queues..."
while read line
  do
    QueueName=`echo $line|cut -d'(' -f2|cut -d')' -f1`
    ProcessQueue=true
    case $QueueName in
    	SYSTEM*) ProcessQueue=false;;
    	*MSGTR.LOG*) ProcessQueue=false;;
    	*MSGTR.TRC*) ProcessQueue=false;;
    	MQMON*) ProcessQueue=false;;
    	AMQ.*) ProcessQueue=false;;
    esac
    	
    if [ $ProcessQueue = true ] ;
    then 
      QueueType=`echo $line|cut -d' ' -f2|cut -d'(' -f2|cut -d')' -f1`      
      #indxL=6
      #indxR=`expr index $line "\)"`
      #QueueS=`expr $indxL + 1`
      #QueueL=`expr $indxR - $QueueS`
      #Queue=`expr substr $line $QueueS $QueueL`
      echo "setting to PUT disabled : $line" >> ${migrationDir}/${qmgr}/queue_status.log
      echo $QueueName
      echo "alter $QueueType('$QueueName') PUT(DISABLED)" | ${su_cmd_pre} "${MQ_HOME}/bin/runmqsc ${qmgr}" > /tmp/"${qmgr}"_disable_queue.txt 
      ret_put_dis=$?
      amqerr=`grep AMQ /tmp/"${qmgr}"_disable_queue.txt`
      grep AMQ8147 /tmp/"${qmgr}"_disable_queue.txt > /dev/null
      ret_grep=$?
      
      
      cat /tmp/"${qmgr}"_disable_queue.txt >> ${migrationDir}/${qmgr}/queue_status.log 
      # todo parse output; not a error AMQ8147: WebSphere MQ object MQMON.FDRESSLE.4BE6A00B2051F402 not found.
      if [ $ret_put_dis != 0 ]; then
      	if [ $ret_grep != 0 ]; then 
      		echo "Warning: ${amqerr} for Queue : $Queue ; continuing"
            #exit $ret_put_dis
      	fi
      fi	
    else
       echo "skipped queue : $line " >> ${migrationDir}/${qmgr}/queue_status.log
    fi
  done < ${Obj_Listfile}
TS=`date +%Y-%m-%d_%H:%M:%S`
echo "$TS - removing work file queue.txt" >> ${migrationDir}/${qmgr}/queue_status.log

echo "--- SUMMARY ---"
echo "Already disable queues:"
wc -l ${migrationDir}/${qmgr}/queue.already.disabled.log
if [ -f ${migrationDir}/${qmgr}/queue.txt ] ; then 
  rm ${migrationDir}/${qmgr}/queue.txt 
fi 
exit 0
