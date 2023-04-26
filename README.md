#mq scripts


#!/usr/bin/ksh
##############################################################################
#
# Script:          mqstart.sh
#
# Usage:          
#
# Description:     start qmgr
#                  
#                  
#
#                  
#                  
#
#                     
#
# 						     checks
##############################################################################
#set -x


# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`
#new
export SCRIPT_NAME=$(basename $0)

WAITTIME=5

# call the common include
. ${HOME}/mqinclude.sh

function start_mqs
{
        QMGR=$1
        rc=0
        [[ "$PLATFORM" = "AIX" ]] && export EXTSHM=ON
        
        count=0
        i=0
        # check if any mq processes are running
        srchstr="( |-m)$QMGR *.*$"
        for process in amqzmuc0 amqzxma0 amqpcsea amqzlaa0 amqzfuma amqzmur0 runmqlsr amqzmgr0
        do
            count=`ps -efww | tr "\t" " " | grep $process | grep -v grep | \
            egrep "$srchstr" | awk '{print $2}'| wc -l`
            if [[ $count -gt 0 ]]
            then
                processes="$processes\n   $process"
                i=1                    
            fi
        done
            
        if [[ $i -eq 0 ]]; then
#clear ipc resources 
#          if [[ -x ${MQ_HOME}/bin/amqiclen ]]; then
#            p_info "Clear all IPC resources"
#            do_as_mqm ${QMGR} "${MQ_HOME}/bin/amqiclen -x -m $QMGR 2>&1"
#          fi

          if [[ -x "${MQ_HOME}/bin/strmqm" ]]; then
#                        exports="export AMQ_BAD_COMMS_DATA_FDCS=TRUE; export AMQ_ADDITIONAL_JSON_LOG=1;"
			echo "Message SEverity:" ${MSGSEVERITY}
                        exports="export AMQ_BAD_COMMS_DATA_FDCS=TRUE;"
			mqversion=$(dspmq -o inst | grep ${QMGR} | awk -F " " '{print $4}' | sed 's/)/(/g' | awk -F "(" '{print $2}' | awk -F "." '{print $1$2$3}')
			if [[ ${mqversion} -ge "910" ]]
            		then
                                if [[ "YES" = "${MSGSEVERITY}" ]];then
                                        exports="$exports export AMQ_DIAGNOSTIC_MSG_SEVERITY=1;"
                    else
                                        exports="$exports export AMQ_DIAGNOSTIC_MSG_SEVERITY=0;"
                                fi
                                if [[ "YES" = "${JSON}" ]]; then
                                        exports="$exports export AMQ_ADDITIONAL_JSON_LOG=1;"
                                fi
            fi
			
#Expiring global units of work
                        AMQ_TRANSACTION_EXPIRY_RESCAN=${AMQ_TRANSACTION_EXPIRY_RESCAN:-$DEF_AMQ_TRANSACTION_EXPIRY_RESCAN}
                        AMQ_XA_TRANSACTION_EXPIRY=${AMQ_XA_TRANSACTION_EXPIRY:-$DEF_AMQ_XA_TRANSACTION_EXPIRY}

 # 0 means off , convert minutes in milliseconds
                        if [[ ${AMQ_TRANSACTION_EXPIRY_RESCAN} -gt 0 ]] ; then

                                AMQ_TRANSACTION_EXPIRY_RESCAN=$(expr ${AMQ_TRANSACTION_EXPIRY_RESCAN} \* 60000)
                                p_info "Setting AMQ_TRANSACTION_EXPIRY_RESCAN=${AMQ_TRANSACTION_EXPIRY_RESCAN}"
                                exports="$exports export AMQ_TRANSACTION_EXPIRY_RESCAN=${AMQ_TRANSACTION_EXPIRY_RESCAN};"
                        else
                                p_info "Disable AMQ_TRANSACTION_EXPIRY_RESCAN"
                        fi
                        if [[ ${AMQ_XA_TRANSACTION_EXPIRY} -gt 0 ]] ; then
                                AMQ_XA_TRANSACTION_EXPIRY=$(expr ${AMQ_XA_TRANSACTION_EXPIRY} \* 60000)
                                p_info "Setting AMQ_XA_TRANSACTION_EXPIRY=${AMQ_XA_TRANSACTION_EXPIRY}"
                                exports="$exports export AMQ_XA_TRANSACTION_EXPIRY=${AMQ_XA_TRANSACTION_EXPIRY};"
                        else
                                p_info "Disable AMQ_XA_TRANSACTION_EXPIRY"
                        fi

            # Start MQSeries Queue-Manager
            p_info "Start queuemanager ${QMGR}"
            echo $exports
        	do_as_mqm ${QMGR} "$exports ${MQ_HOME}/bin/strmqm $QMGR 2>&1|sed 's/^/          /'" # su mqm -c (without - ; to preserver environment )
			erg=$?
			case ${erg} in
				0)   msg="Queue manager started (MQCC_OK)";;
				1)   msg="Warning partial completion (MQCC_WARNING)";;
				2)   msg="Call failed (MQCC_FAILED)";;
				3)   msg="Queue manager being created";;
				5)   msg="Queue manager running";;
				16)  msg="Queue manager does not exist";;
				23)  msg="Log not available";;
				24)  msg="A process that was using the previous instance of the queue manager has not yet disconnected.";;
				30)  msg="A standby instance of the queue manager started. The active instance is running elsewhere";;
				31)  msg="The queue manager already has an active instance. The queue manager permits standby instances.";;
				39)  msg="Invalid parameter specified";;
				43)  msg="The queue manager already has an active instance. The queue manager does not permit standby instances.";;
				47)  msg="The queue manager already has the maximum number of standby instances";;
				49)  msg="Queue manager stopping";;
				58)  msg="Inconsistent use of installations detected";;
				62)  msg="The queue manager is associated with a different installation";;
				69)  msg="Storage not available";;
				71)  msg="Unexpected error";;
				72)  msg="Queue manager name error";;
				74)  msg="The WebSphere MQ service is not started.";;
				91)  msg="The command level is outside the range of acceptable values.";;
				92)  msg="The queue manager's command level is greater or equal to the specified value.";;
				100) msg="Log location invalid";;
				119) msg="User not authorized to start the queue manager";;
				*)   msg="unknown return code";;
			esac
			case ${erg} in
				0|3|5)  msg=""
				        p_info_ts "wait ${WAITTIME} sec to check ${QMGR}"
				        sleep ${WAITTIME}
				        if [[ $(dspmq -m $QMGR | grep "STATUS(Running)"| wc -l) -eq 1 ]]; then 
               				p_info_ts "Status Qmgr $QMGR : STATUS(Running)"
               			else
#               			    call_logger CRONMON 2 "second time mqstart $QMGR"
#new
                                call_logger CRONMON 2 ${SCRIPT_NAME} "second time mqstart $QMGR"
               			    p_info_ts "2. Start queuemanager ${QMGR}"
            				do_as_mqm ${QMGR} "$exports ${MQ_HOME}/bin/strmqm $QMGR 2>&1|sed 's/^/          /'" # su mqm -c (without - ; to preserver environment )
							erg=$?
               			    rc=$erg
               			fi
              			;;
				*)  	 call_logger CRONMON 3 ${SCRIPT_NAME} "mqstart $QMGR $msg"
#				call_logger CRONMON 3 "mqstart $QMGR : $msg"
				    	rc=$erg
				    	p_error_ts "Return code strmqm : $erg : $msg";;
			esac
          fi
        else
		   p_warning_ts "Exisiting WMQ-process(es) for $QMGR found: $processes"
#          echo "$MQSCRIPT : Warning ! Exisiting WMQ-process(es) for $QMGR found: $processes"
          rc=2
        fi
}

function start_broker
{
  qmgrName=$1
        
  # call the IIB Tools script to start an Integration Node by the name of its Queue Manager
  iibstartnode -qmgr $qmgrName
}

function invoke {
   QMGR=$1
   MODE=$2
   rc=0
   
   if [[ "SINGLE" = "${MODE}" || "YES" = "${STRTWMQ}" ]];then
      p_info_ts "Starting queuemanager ${QMGR}"
      start_mqs ${QMGR}      
      
      if [[ "YES" = "${STRTBRK}" ]]
      then
         p_info_ts "Starting broker ${QMGR}"
         start_broker ${QMGR}    
      fi    

      p_info_ts "Summary RC for ${QMGR} $rc"
      
   else
      p_info_ts "Skip start of queuemanager ${QMGR} by config"
#      echo "Skip start of queuemanager ${QMGR} by config"
   fi 
}


# called befor do all other 
function invokestartup {
	# get time sync status
	p_info "\n$(echo "q"|ntpq -p)"
	
	i=0
	# check if any mq processes are running
	for process in amqzmuc0 amqzxma0 amqpcsea amqzlaa0 amqzfuma amqzmur0 runmqlsr amqzmgr0
	do
	    count=`ps -efww | tr "\t" " " | grep $process | grep -v grep | awk '{print $2}'| wc -l`
	    if [[ $count -gt 0 ]]
	    then
	        i=1                    
	    fi
	done
	    
	if [[ $i -eq 0 ]]; then
	# clear all ipc resources
	  if [[ -x ${MQ_HOME}/bin/amqiclen ]]; then
	    p_info_ts "Clear all MQ-IPC resources"
	    ${su_cmd_pre}"${MQ_HOME}/bin/amqiclen -x 2>&1"
	  fi
	fi
}

###############################################################################
# call the common routine for all queue managers
. ${HOME}/mqinvokeforqmgrs.sh
