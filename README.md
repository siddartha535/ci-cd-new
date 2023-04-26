#!/bin/ksh
##############################################################################
#$Id: mqcheck.sh,v 1.24 2011/10/25 15:56:04 EMEA\lreckru Exp $
#
# Script:         mqcheck.sh
#
# Usage:          mqcheck.sh  <queue manager>
#
# Description:
#
# Remarks:
#
# History:         
#       Name:        checkmq
#       Funktion:    ueberpruefen der Queuemanager- und WBIMB-Broker Prozesse
#       Autor:       Hartmut Barkawitz
#       Datum:       17.12.2004
#
#       geaendert:   Hartmut Barkawitz     09. Januar 2006
#                    Englische Ausgabetexte
#
#       2011-08-04   lreckru    initial new version, new function chk_service
#       2012-07-18   lreckru    add new function "check_fs_and_env" 
#       2014-11-17   sanzmaj    multiversion
#                    phutzel    checks
#       2014-12-18   lreckru    recreate output  
#       2015-08-13   lreckru    change add some return codes 
#                               
##############################################################################
#set -x

# Rc values - 1 parametererror, 2 warning, 3 error , 4 error
 
# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`

# call the common include
. ${HOME}/mqinclude.sh

if [[ $# -eq 0 ]]; then
        echo "/opt/mw/mqm/bin/mqcheck.sh <QMGR NAME> | -all"
        exit 1;
fi

function chk_service
{
# check running

   runall=yes
   OBJCLASS=$1
   FILTER1=$2
   FILTER2=$3
   OBJECT=$4

   RUNIST=$(echo "DISPLAY ${OBJCLASS}(${OBJECT}) where (CONTROL EQ QMGR) ALL" |  do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null |grep ${FILTER1} | grep ${FILTER2} | grep -v dis| sed -e 's/(/ /' -e 's/)/ /' |awk '{print $2}')

# set new parameters
   OBJCLASS=$5
   FILTER1=$6
   FILTER2=$7

   RUNSOLL=`echo "DISPLAY ${OBJCLASS}(${OBJECT}) where (CONTROL EQ QMGR) ALL" |  do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null |grep ${FILTER1} | grep ${FILTER2} | grep -v dis| sed -e 's/(/ /' -e 's/)/ /' |awk '{print $2}'`

   for OBJECT1 in $RUNSOLL
   do
   #   echo "DIS $1 (${OBJECT1}) " |  do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"  2>/dev/null | grep ${OBJECT1} | grep -v STDERR |grep -v AMQ | grep -v "DIS $1 (${OBJECT1})"
      if [ `echo $RUNIST | grep $OBJECT1 | wc -l` -ne 1 ]; then
                p_warning "$(printf "%-35s %s" ${OBJECT1} 'NOT RUNNING')"
                  if [[ $q_rc -lt 2 ]];then
                                q_rc=2
                                  fi
              else
                  p_info "$(printf "%-35s %s" ${OBJECT1} RUNNING)"
              fi
        done
}


function put_get
{
 
   mqstext="TestNachricht"
   queue=SYSTEM.DEFAULT.LOCAL.QUEUE
   persist=-ap
   mqsintext=/tmp/mqsintext.txt
   mqsouttext=/tmp/mqsouttext.txt
   username=mqm
   mqsuser="-U ${username}"
   echo ${mqstext}>${mqsintext}

   p_info "Check write/read a message"
   # clear all message from queue    
    do_as_mqm ${qmgr} "${MQTOOLS_HOME}/bin/q -I ${queue} -m ${qmgr} -F ${mqsouttext} ${mqsuser} >/dev/null 2>&1" 
   rc=$?
   [[ -f ${mqsouttext} ]] && rm ${mqsouttext}
   if [ $rc -ne 0 ] ; then
   
      ##
      ## amqsget & amqsput
      ## 
      p_info "q command not work, trying with amqsget and amqsput (needs 30 sec)"
      # clean messages with amqsget
       do_as_mqm ${qmgr} "${MQ_HOME}/samp/bin/amqsget ${queue} ${qmgr} >/dev/null 2>&1"  

      # write persistent message to queue
       do_as_mqm ${qmgr} "${MQ_HOME}/samp/bin/amqsput ${queue} ${qmgr} < ${mqsintext}   >/dev/null 2>&1" 
      rc=$?
      
      if [[ ${rc} -eq 0 ]];then
         # read persistent message from queue
          do_as_mqm ${qmgr} "${MQ_HOME}/samp/bin/amqsget ${queue} ${qmgr} | grep \"^message \" | sed 's/message <//g;s/>$//g' > ${mqsouttext} 2>&1"
         rc=$? 
         
            if [[ ${rc} -eq 0 ]];then
               # compare write and read message
               diff ${mqsouttext} ${mqsintext}
               rc=$?
               if [[ ${rc} -eq 0 ]];then
                  p_info "Message write/read sucessfull."
               else
                  p_info "The read message is not equal to the written message."
               fi
               rm ${mqsouttext}
            else
               p_error "Can't read the persistent message from the queue."
            fi
            rm ${mqsintext}
         
      else
         p_error "Can't write the persistent message in the queue."        
      fi
      
   else
   
      ##
      ## q command
      ##
      
      # write persistent message to queue
       do_as_mqm ${qmgr} "${MQTOOLS_HOME}/bin/q -o ${queue} -m ${qmgr} -F ${mqsintext} ${mqsuser} ${persist} >/dev/null 2>&1"
      rc=$?
      
      if [[ ${rc} -eq 0 ]];then
         # read persistent message from queue
          do_as_mqm ${qmgr} "${MQTOOLS_HOME}/bin/q -I ${queue} -m ${qmgr} -F ${mqsouttext} ${mqsuser} >/dev/null 2>&1"
         rc=$?
         
            if [[ ${rc} -eq 0 ]];then
               # compare write and read message
               diff ${mqsouttext} ${mqsintext}
               rc=$?
               if [[ ${rc} -eq 0 ]];then
                  p_info "Message write/read sucessfull."
               else
                  p_info "The read message is not equal to the written message."
               fi
               rm ${mqsouttext}
            else
               p_error "Can't read the persistent message from the queue."
            fi
            rm ${mqsintext}
         
      else
         p_error "Can't write the persistent message in the queue."       
      fi
      
   fi
q_rc=$rc  
return $q_rc
}


function check_fs_and_env
{

# checking for variables 'server =' and 'scan_group ='
#grep "[e|u][r|p] = " /etc/tlmagent.ini 1>2 2>/dev/null
#if [[ $? -eq 0 ]];then
 #       rc=0
  #      p_info "check for ILMT variable 'server =' and 'scan_group ='"
   #     grep "[e|u][r|p] = " /etc/tlmagent.ini | sed 's/^/          /'
#else
#        p_warning "check for ILMT variable 'server =' and 'scan_group ='"
 #       rc=1
#fi
if [[ `id -u -n` = "root" ]];then
	p_info "pvscan"
	pvscan|sed 's/^/        /'
else
	p_info "pvscan - only if script called as root"
fi

#echo Sample output:
# PV /dev/sdd    VG data2vg   lvm2 [234.00 GiB / 234.00 GiB free]
# PV /dev/sdc    VG data1vg   lvm2 [101.00 GiB / 75.81 GiB free]
# PV /dev/sdb1   VG vgswap    lvm2 [3.97 GiB / 64.00 MiB free]
# PV /dev/sda2   VG vg00      lvm2 [19.72 GiB / 3.25 GiB free]
# Total: 4 [358.68 GiB] / in use: 4 [358.68 GiB] / in no VG: 0 [0   ]
#
# important 
# data2vg   must be on   /dev/sdd and 
# data1vg   must be on   /dev/sdc 
#
q_rc=$rc  
return $q_rc

}

##
##  INVOKE
##  

function invoke {
   rc=0
   
syntax() {
  echo "Syntax : ${MQSCRIPT}  <queue manager>|-all "
  ende $1
}

ende() {
  p_error_ts "Qmgr: ${qmgr}"
  p_linie
  exit $q_rc
}

# Wurde Name des Queue-Managers uebergeben ?
if [ -z "$1" ] ; then
  syntax $1
else
   typeset -u qmgr=$1
fi

if [ ! -d ${MQ_LOG}/${qmgr} ] ; then
   p_warning "There is no queue manager $bold${qmgr}$off created on this server !!!"
   if [[ $q_rc -lt 2 ]];then
               q_rc=2
   fi
   return $q_rc
fi

case "$PLATFORM" in
 "HP-UX")
    if [ ! -d ${MQ_DATA}/${qmgr} ] ; then
       p_warning "The queue manager $bold${qmgr}$off is not running on this node !!!"
       if [[ $q_rc -lt 2 ]];then
               q_rc=2
       fi
       return $q_rc
    fi
    ;;
 "SunOS")
    if [ ! -d ${MQ_DATA}/${qmgr} ] ; then
       p_warning "The queue manager $bold${qmgr}$off is not running on this node !!!"
       if [[ $q_rc -lt 2 ]];then
               q_rc=2
       fi
       return $q_rc
    fi
    ;;
 "AIX")
    QmgrAktiv="N"
    for VolGr in `lsvg`
    do
      if [ $(lsvg -l  $VolGr | grep ${qmgr} | wc -l ) -ne 0 ] ; then
         if [ $( lsvg  -o  |  grep $VolGr | wc -l ) -ne 1 ] ; then
            p_warning "The volumegroup  ${VolGr}         is not active   !!!"
            p_warning "The queue manager $bold${qmgr}$off is not running on this node !!!"
            if [[ $q_rc -lt 2 ]];then
               q_rc=2
            fi
            return $q_rc
         else
            QmgrAktiv="Y"
         fi
      else
         if [ $(lsvg -l  $VolGr | grep /MQHA | wc -l ) -ne 0 ] ; then
            if [ -d ${MQ_DATA}/${qmgr} ]; then
               QmgrAktiv="Y"
            fi
         fi
      fi
    done

    if [ "$QmgrAktiv" = "N" ]
    then
       p_warning "The volumegroup  ${VolGr}         is not active   !!!"
       p_warning "The queue manager $bold${qmgr}$off is not running on this node !!!"
       if [[ $q_rc -lt 2 ]];then
           q_rc=2
       fi
       return $q_rc
    fi
    ;;
 "Linux")
    if [ ! -d ${MQ_DATA}/${qmgr} ] ; then
       p_warning "The queue manager $bold${qmgr}$off is not running on this node !!!"
       if [[ $q_rc -lt 2 ]];then
           q_rc=2
       fi
       return $q_rc
    fi
    ;;
*)
    ;;
esac


   #echo $cls$off
	vers=$(do_as_mqm ${qmgr} "${MQ_HOME}/bin/dspmqver -f 2")
    p_info "MQSeries $vers "
	p_info "Queue Manager:     ${bold}${qmgr}${off}"
    MQSTATUS=""
    typeset -u MQSTATUS=`do_as_mqm ${qmgr} "${MQ_HOME}/bin/dspmq -m ${qmgr}" | awk ' /STATUS/ { string = substr($2, 8,7) ; print string } ' `
    p_info "$(printf "%-35s %s" 'Queuemanager status:' $MQSTATUS)"

   
   if [ $mqversion -ge 800 ] ; then
      #  MQ 8 and higher -->  MQ8_processes='amqzxma0|amqzmuc0|amqzmur0|amqrrmfa'
      ANZPROC1=4
      ANZPROC=`$PS $PSPARM|$GREP -v grep|$EGREP -e 'amqzxma0|amqzmuc0|amqzmur0|amqrrmfa' | grep -wc ${qmgr}`
   else
      #  MQ 7 --> MQ7_processes='amqzxma0|amqzmuc0|amqzmur0|amqzdmaa|amqrrmfa'
      ANZPROC1=5
      ANZPROC=`$PS $PSPARM|$GREP -v grep|$EGREP -e 'amqzxma0|amqzmuc0|amqzmur0|amqzdmaa|amqrrmfa' | grep -wc ${qmgr}`
   fi
   
  
	# check qmgr processes
	
	if [ $ANZPROC -ne $ANZPROC1 ] ; then
		p_error "$(printf "%-35s %s" 'necessary qmgr processes' 'NOT RUNNING')"
		q_rc=3
	else
		p_info "$(printf "%-35s %s" 'necessary qmgr processes' 'RUNNING')"
	
		if [ $($PS $PSPARM | grep -v grep | grep amqpcsea | grep ${qmgr} | wc -l ) -ne 1 ] ; then
	    	p_error "$(printf "%-35s %s" Commandserver:  'NOT RUNNING')"
	    	q_rc=3
		else
	    	p_info "$(printf "%-35s %s" Commandserver:  'RUNNING')"
		fi
	
		if [ $($PS $PSPARM | grep -v grep | grep runmqchi | grep ${qmgr} | wc -l ) -ne 1 ] ; then
	    	p_error "$(printf "%-35s %s" Channelinitiator:  'NOT RUNNING')"
	    	if [[ $q_rc -lt 2 ]];then
	           q_rc=2
	    	fi
		else
	    	p_info "$(printf "%-35s %s" Channelinitiator:  'RUNNING')"
		fi
	
	      p_info "Checking Listener "
	      OBJECT="LISTENER.*"
	      OBJCLASS=LSSTATUS
	      FILTER1=LISTENER
	      FILTER2="STATUS(RUNNING)"
	      OBJCLASS1=LISTENER
	      FILTER11=LISTENER
	      FILTER12="CONTROL(QMGR)"
	      chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12
	
	      p_info "Check Trigger"
	      OBJCLASS=SVSTATUS
	      FILTER1=TRIGGER
	      FILTER2="STATUS(RUNNING)"
	      OBJECT="TRIGGER.*"
	      OBJCLASS1=SERVICE
	      FILTER11=TRIGGER
	      FILTER12="CONTROL(QMGR)"
	      chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12

mqxrcount=`rpm -qa | grep MQSeriesXRService | wc -l`
if [[ $mqxrcount -ge 1 ]];then

	      p_info "Check MQXR Service "
          OBJCLASS=SVSTATUS
          FILTER1=MQXR
          FILTER2="STATUS(RUNNING)"
          OBJECT="SYSTEM.MQXR.*"
          OBJCLASS1=SERVICE
          FILTER11=MQXR
          FILTER12="CONTROL(QMGR)"
          chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12
fi

fi  # end of else part of if [ $ANZPROC -ne $ANZPROC1 ]


p_info "Checking MQ-Agents"
  for MQ_AGENT in mqagentadmin mqagentmonitor
  do
   if [ -f ${MQTOOLS_HOME}/bin/${MQ_AGENT} ]; then
if [ "$($PS ${PSPARM} | grep -v grep | grep -c "${MQ_AGENT} -qmgr ${qmgr}")" -ne "1" ]
      then
         p_error "$(printf "%-35s %s" ${MQ_AGENT}: 'NOT RUNNING')"
         q_rc=3
      else
         p_info "$(printf "%-35s %s" ${MQ_AGENT}: 'RUNNING')"
      fi
   else
         p_error "There is no ${MQ_AGENT} installed on this server !!!"
         q_rc=3
   fi
  done
  
# call put/get check
put_get

# check FS and env
check_fs_and_env

####new
#ignore the check as the initial log backup is not taken anymore
#filename=${MW_BACKUP_DIR}/${myHostname}-${qmgr}-initial-logs.tar.gz
#if [ ! -f ${filename} ]; then
#        p_warning "Initial log backup files not found for QMGR: ${qmgr}"
#        if [[ $q_rc -lt 2 ]];then
#                q_rc=2
#        fi
#else
#        p_info "Initial log backup files found for QMGR: ${qmgr}"
#fi

filename=${MW_BACKUP_DIR}/${myHostname}-${qmgr}-initial-data.tar.gz
if [ ! -f ${filename} ]; then
        p_warning "Initial data backup files not found for QMGR: ${qmgr}"
        if [[ $q_rc -lt 2 ]];then
                q_rc=2
        fi
else
        p_info "Initial data backup files found for QMGR: ${qmgr}"
fi

#####

# check variable LD_LIBRARY_PATH in mqinclude.sh 
chk_variable_LD_LIBRARY_PATH

if [ "$q_rc" -ne "0" ] ; then 
     p_info "Returncode for ${qmgr} = $q_rc"
    if [[ "$g_rc" -lt "$q_rc" ]];then
         g_rc=$q_rc
     fi
fi
 return ${rc}
}
## end Invoke()

##
## Main
###################################################################

# call the common routine for all queue managers
g_rc=0
 . ${HOME}/mqinvokeforqmgrs.sh
if [[ $g_rc -lt $q_rc ]];then
        p_info_ts "Returncode for all = $g_rc"
     fi
return $g_rc
