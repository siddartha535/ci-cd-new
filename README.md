#!/bin/ksh
##############################################################################
#
# Script:          mqsaveqmgr.sh
#
# Usage:          
#
# Description:    call saveqmgr for config and OAM 
#                  
#                     
#
# Remarks:         
#                  
#                  
#                  
#                  
#
# History:         2012-08-04   lreckru     initial version
#                  2014-04-03   lreckru     add DATE variable (line 35)
#                  2014-08-18   lreckru     add calling logger
#                  2014-11-27   sanzmaj     multiversion
#                               phutzel     invoke, checks
#
##############################################################################
#set -x
#new
export SCRIPT_NAME=$(basename $0)
# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`

# call the common include
. ${HOME}/mqinclude.sh


function invoke {
   
   qmgr=$1
   DATE=`date +"%Y%m%d-%H%M"`
   rc=0
   QMGR_FILE=${MQTOOLS_DATA}/config/${qmgr}.$DATE.mqsc
   AUT_FILE=${MQTOOLS_DATA}/config/${qmgr}.$DATE.aut
  
   # check/create file system 
   if [[ ! -d  ${MQTOOLS_DATA}/config ]]; then
      mkdir -p ${MQTOOLS_DATA}/config
      chown mqm:mqm ${MQTOOLS_DATA}/config
   fi


   # check if qmgr is running    
   pid=`ps -efww | grep "amqzxma0 -m ${qmgr}" | grep -v grep | awk '{ print $2 }'`
   if [[ "${pid}" = "" ]]; then
      p_error "skip ${qmgr} - it is not running!"
#      call_logger CRONMON 0 "skip ${qmgr} - it is not running!"
#new
       call_logger CRONMON 0 ${qmgr} "${SCRIPT_NAME}:skip ${qmgr} - it is not running!"
      rc=2
   else
   
      if [ $mqversion -ge 710 ]; then
         ### 7.1 new command :  dmpmqcfg  
         # *** new funtionality dump config of a remote queue manager : -> dmpmqcfg -m R_QM -a -c "DEFINE CHANNEL(YYYY.DEF.SVRCONN) CHLTYPE(CLNTCONN) CONNAME('abc.def.com(1414)') SSLCIPH(RC4_SHA_US)"
         ${su_cmd_mqm_keep_env}"${MQ_HOME}/bin/dmpmqcfg -m ${qmgr} -a"                            >  ${QMGR_FILE}      
         [[ $? -eq 0 ]] && ${su_cmd_mqm_keep_env}"${MQ_HOME}/bin/dmpmqcfg -m ${qmgr} -o setmqaut" >  ${AUT_FILE}        
         rc=$?
         # call saveqmgr for config and OAM      
         #${su_cmd_mqm_keep_env}"${MQTOOLS_HOME}/bin/saveqmgr -m ${qmgr} -f ${QMGR_FILE} -z ${AUT_FILE}"
         #rc=$?         
      else 
         # call saveqmgr for config and OAM      
         ${su_cmd_mqm_keep_env}"${MQTOOLS_HOME}/bin/saveqmgr -m ${qmgr} -f ${QMGR_FILE} -z ${AUT_FILE}"
         rc=$?
      fi  
       
      # if configfile not exist return errorcode 2  
      if [[ ! -f ${QMGR_FILE} ]]; then
         p_error "No config file '${QMGR_FILE}' saved for queuemanager ${qmgr}"
#         call_logger CRONMON 3 "No config file '${QMGR_FILE}' saved for queuemanager ${qmgr}"
#new
          call_logger CRONMON 3 ${qmgr}_${QMGR_FILE} "${SCRIPT_NAME}:No config file '${QMGR_FILE}' saved for queuemanager ${qmgr}"
         rc=2        
      fi
      if [[ ! -f ${AUT_FILE} ]]; then
         p_error "No aut. file '${AUT_FILE}' saved for queuemanager ${qmgr}"
#         call_logger CRONMON 3 "No aut. file '${AUT_FILE}' saved for queuemanager ${qmgr}"
#new
          call_logger CRONMON 3 ${qmgr}_${AUT_FILE} "${SCRIPT_NAME}:No aut. file '${AUT_FILE}' saved for queuemanager ${qmgr}"
         rc=2     
      fi
     
   fi
   echo "RC for ${qmgr} = $rc"
   return ${rc}
}


#####################################################################################

# call the common routine for all queue managers
. ${HOME}/mqinvokeforqmgrs.sh
