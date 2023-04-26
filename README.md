#!/bin/bash
##############################################################################
#
# Script:          mqinvokeforqmgr.sh
#
# Usage:          
#
# Description:     Checks if qmgr is running, adds vars for multiversion and invoke
#                  
#                  
#
# Remarks:         
#                  
#                  
#
# History:         2012-08-08   fdress     initial version
#                  2013-09-23   lreckru    add some loggin infos with time stamp
#				       2014-04-03   lreckru    add unset "Expiring global units of work" variables
#
##############################################################################
#set -x
#new
export SCRIPT_NAME=$(basename $0)

function checkAndInvoke {
#set -x
   qmgr=$1
   MODE=$2
   INCR=$3
    p_info_ts "Performing checks for queuenanager $1 with mode $2 - look also at ${MQLOGNAME}-${qmgr}.log " >>${MQLOGNAME}.log
	(  
	    
	    echo ""
	    p_info_ts "Processing queue manager ${qmgr}"
	    
       # check if queue manager is there and accessible
	    if [[ ${MQTOOLS_CHECK:-YES} = "YES" && ! -d ${MQ_LOG}/${qmgr}/active ]] ; then
			p_error "No access to queue manager file system ${MQ_LOG}/${qmgr}/active"
#			call_logger CRONMON 2 "No access to queue manager file system ${MQ_LOG}/${qmgr}/active"
#new
            call_logger CRONMON 2 ${qmgr} "${SCRIPT_NAME}:No access to queue manager file system ${MQ_LOG}/${qmgr}/active"
	    else
         # clear variables used in locals files
   		unset STRTWMQ STRTBRK LOGCLEANUP MQUSER TIMEO AMQ_TRANSACTION_EXPIRY_RESCAN AMQ_XA_TRANSACTION_EXPIRY

			# set defaults
			MQUSER=mqm

			# check config file
		   if [[ ${MQTOOLS_CHECK:-YES} = "YES" && ! -f ${MQTOOLS_ETC}/locals_${qmgr} ]]; then
               p_error "Queue Manager is on this system for ${qmgr} not configured !"
               rc=1
               return
         fi
		    
		   # source config file
         if [[ -f ${MQTOOLS_ETC}/locals_${qmgr} ]]; then
		       . ${MQTOOLS_ETC}/locals_${qmgr}
         fi


         # Take vars for MQ Multiversion
         #. ${MQMHOME}/include/set_mqenv_vars.sh ${qmgr}  2>/dev/null  &&
         #MQ_HOME=$MQMDIR                                            ||
         #p_error_ts "Warning Multiversion include files were not executed correctly" >> ${MQLOGNAME}.log
     
         mqsetenv -m ${qmgr}

         # invoke main function from called script
                 invoke ${qmgr} $MODE ${INCR}
	    fi
       
    ) | ( ${su_cmd_pre}"tee -a ${MQLOGNAME}-${qmgr}.log" 2>/dev/null ) # ignore following output 'stty: standard input: Invalid argument'
}

############################### Main #######################################

# remove last extention from script name for logging file name"abcd.*" -> "abc"
MQLOGNAME=$MQTOOLS_LOG/${MQSCRIPT%.*}

if [[ -z "$1" ]]; then
      p_error_ts " No Parameter  -all or {qmgrname}" >>${MQLOGNAME}.log
      exit 1
fi

p_info_ts "Start script ${MQSCRIPT}" >>${MQLOGNAME}.log

$(type invokestartup 2>/dev/null|grep -q 'is a function')
	  if [[ "$?" -eq "0" ]];then
	  	p_info_ts "call function invokestartup" >>${MQLOGNAME}.log
	  	invokestartup | ${su_cmd_pre}"tee -a ${MQLOGNAME}.log"
      fi
      
if [[ "$1" = "-all" ]]; then
      p_info_ts "Invoked for all queuemanagers" >>${MQLOGNAME}.log
      # comment while read and use instead for --- with the first one I have seen some issues in our suse test server
      #ls -1 ${MQTOOLS_ETC}/locals_* | while read LOCALSFILE
      for LOCALSFILE in `ls -1 ${MQTOOLS_ETC}/locals_*`      
      do
       # just filename
          FILENAME=`basename ${LOCALSFILE}`
          # extract name of qmgr
          qmgr=${FILENAME#locals_}
          #   p_info_ts "Invoke ${qmgr} from all" >>${MQLOGNAME}.log 
          if [[ "$2" = "-increment" ]]; then
                arg3=$2
                  checkAndInvoke ${qmgr} MULTIPLE ${arg3}
          fi
          if [[ -z "$2" ]]; then
          checkAndInvoke ${qmgr} MULTIPLE
          fi

      done       
 else
      # invoke for passed queue manager 
      p_info_ts "Invoked for single queuemanager" >>${MQLOGNAME}.log
        if [[ "$2" = "-increment" ]]; then
                arg4=$2
                  checkAndInvoke $1 SINGLE ${arg4}
          fi
          if [[ -z "$2" ]]; then
      checkAndInvoke $1 SINGLE
 fi


 fi
 p_info_ts "End of script ${MQSCRIPT}" >>${MQLOGNAME}.log
