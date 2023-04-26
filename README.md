#!/usr/bin/ksh
##############################################################################
#$Id: mqstop.sh,v 1.4.2.2 2011/09/26 10:46:50 EMEA\lreckru Exp $
#
# Script:          mqstop.sh
#
# Usage:          
#
# Description:     check processes and stop qmgr
#                  
#                  
#
#                  
#                  
#
#                     
#
# Remarks:         
#                  
#                  
#                  
#                
#
# History:         2012-08-04   fdressle     initial version
#                  2014-11-27   sanzmaj      multiversion
#                               phutzel      invoke, checks
#                  2015-02-17   jduelli      function stop_agents
#                  2015-05-20	lreckru      change replace variable su_cmd_mqm_keep_env, su_cmd_mqm_pre with fkt do_as_mqm
##############################################################################
#set -x

# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`

# call the common include
. ${HOME}/mqinclude.sh

# Time to wait between checks
SLEEPTIME1=1
# Time to wait between immediate and brutal end
SLEEPTIME2=1

function stop_mqs
{
    qmgr=$1
    count=0
    i=0
    
    # check if any process are running
    srchstr="( |-m)${qmgr} *.*$"
    for process in amqzmuc0 amqzxma0 amqfcxba amqfqpub amqpcsea amqzlaa0 \
                           amqzlsa0 runmqtrm runmqchi runmqlsr amqcrsta amqrrmfa amqrmppa \
                           amqzfuma amqzdmaa amqzmuf0 amqzmur0 amqzmgr0 MQXR   
        do
            count=`ps -efww | tr "\t" " " | grep $process | grep -v grep | \
            egrep "$srchstr" | awk '{print $2}'| wc -l`
            if [[ $count != 0 ]];then
                i=`expr $i + 1`                    
             fi
        done
        
    if [[ $i -eq 0 ]];then 
	p_info_ts "No running queue manager processes found."
#        echo "$MQSCRIPT : Info ! No running queue manager processes found."
    else 
        for severity in immediate brutal
        do
              # End the queue manager in the background to avoid
              # it blocking indefinitely. Run the TIMEOUT timer 
              # at the same time to interrupt the attempt, and try a
              # more forceful version. If the brutal version fails, 
              # nothing more can be done here.
              
              defpsistno=$(echo "display ql(*) where(DEFPSIST EQ NO)" | do_as_mqm ${qmgr} "runmqsc ${qmgr}" | grep QUEUE | grep -v SYSTEM | wc -l)
              if [[ ${defpsistno} -ne 0 ]]; then
                  queuenames=$(echo "display ql(*) where(DEFPSIST EQ NO)" | do_as_mqm ${qmgr} "runmqsc ${qmgr}" | grep QUEUE | grep -v SYSTEM | awk -F " " '{print $1}')
                  for queue in ${queuenames}
                  do
                      npmclass=$(echo "display ${queue} NPMCLASS" | do_as_mqm ${qmgr} "runmqsc ${qmgr}" | grep "NPMCLASS(" | sed 's/(/)/' | awk -F ")" '{print $2}')
                      if [[ ${npmclass} = "HIGH" ]]; then
                          SLEEPTIME2=1200
                          break
                      fi
                  done
              fi
              
              
     	      p_info_ts "Attempting ${severity} end of queue manager '${qmgr}'"
              case $severity in
            
              immediate)
                # Minimum severity of endmqm is immediate which severs connections.
                   if [[ -x ${MQ_HOME}/bin/endmqm ]] ;then
                       # Stop Queue-manager
                       ( do_as_mqm ${qmgr} "${MQ_HOME}/bin/endmqm -i ${qmgr} " 2>/dev/null | grep MQSeries )&
                    else
		              p_warning_ts "endmqm not executable, not all qmgr processes are stopped !"
                      rc=4
                    fi
                ;;
            
              brutal)
                # This is a forced means of stopping queue manager processes.
            
                srchstr="( |-m)${qmgr} *.*$"
                for process in amqzmuc0 amqzxma0 amqfcxba amqfqpub amqpcsea amqzlaa0 \
                           amqzlsa0 runmqtrm runmqchi runmqlsr amqcrsta amqrrmfa amqrmppa \
                           amqzfuma amqzdmaa amqzmuf0 amqzmur0 amqzmgr0 MQXR   
                do
                  ps -efww | tr "\t" " " | grep $process | grep -v grep | \
                     egrep "$srchstr" | awk '{print $2}'| \
                        xargs kill -9 > /dev/null 2>&1
                done
            
              esac
            
              TIMED_OUT=yes
              SECONDS=0
              while (( $SECONDS < ${TIMEO:-55} ))
              do
               TIMED_OUT=yes
               i=0
               while [[ $i -lt 5 ]]
               do
                 # Check for execution controller termination
                 srchstr="( |-m)${qmgr} *.*$"
                 cnt=`ps -efww | tr "\t" " " | grep amqzxma0 | grep -v grep | \
                   egrep "$srchstr" | awk '{print $2}' | wc -l `
                 i=`expr $i + 1`
                 sleep $SLEEPTIME1
                 if [[ $cnt -eq 0 ]];then
                   TIMED_OUT=no
                   break
                 fi
               done
            
               if [[ ${TIMED_OUT} = "no" ]];then
                 break
               fi
            
#               echo "$MQSCRIPT : Info ! Waiting for ${severity} end of queue manager '${qmgr}'"
	       p_info_ts "Waiting for ${severity} end of queue manager '${qmgr}'"
               sleep $SLEEPTIME2
              done # timeout loop
                        
        done # next phase       
    fi
}

function stop_agents
{
	qmgr=$1
	count=0
	cnt=0

	for process in mqagentadmin mqagentmonitor
	do
		count=`ps -efww | tr "\t" " " | grep $process | grep $qmgr | grep -v grep | awk '{print $2}'| wc -l`
		cnt=`expr $cnt + $count`
	done

	if  [[ $cnt > 0 ]]; then
		p_info_ts "Stopping ${cnt} running agents of queuemanager ${qmgr}"
		for process in mqagentadmin mqagentmonitor
		do
			ps -efww | tr "\t" " " | grep $process | grep $qmgr | grep -v grep | awk '{print $2}'| xargs kill -9 > /dev/null 2>&1
		done
	else
		p_info_ts "Found no running agents for queuemanager ${qmgr}"
	fi
}

function stop_broker
{
        qmgrName=$1
        
		  # call the IIB Tools script to start an Integration Node by the name of its Queue Manager
		iibstopnode -qmgr $qmgrName
}

function invoke {
   qmgr=$1
   rc=0
   
    if [[ "YES" = "${STRTBRK}" ]]
      then
         p_info_ts "Stopping broker ${qmgr}"
         stop_broker ${qmgr}    
      fi
  
   p_info_ts "Stopping queuemanager ${qmgr}"
   stop_mqs ${qmgr} 

   # TODO run for all qmgrs automatically?
   stop_agents ${qmgr}   
      
}
##############################################################################
# call the common routine for all queue managers
MQTOOLS_CHECK=NO
. ${HOME}/mqinvokeforqmgrs.sh
