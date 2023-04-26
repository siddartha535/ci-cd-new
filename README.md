#!/bin/ksh
 ##############################################################################
 #
 # Script:         mqcheckqmgr.sh
 #
 # Usage:          mqcheckqmgr.sh  <queue manager | -all> 
 #
 # Description:    check Queuemanager Processes 
 #                  
 #                     
 #
 # Remarks:         
 #                    
 #
 # History:
 #
 #      2011-08-04   lreckru     new version
 #      2014-11-27   sanzmaj     multiversion
 #                   phutzel     invoke checks
 #      2015-05-12   lreckru     add check variable LD_LIBRARY_PATH
 #      2016-04-18   lreckru     add HP-OV message calls, different sleep values
 #      2019-07-02   mashiki     updated to use new version of queueopenkeeper program
 ##############################################################################
 #set -x

 # get full path to this script
 HOME=`readlink -f $0`
 HOME=`dirname $HOME`
 #new
 export SCRIPT_NAME=$(basename $0)
 # call the common include
. ${HOME}/mqinclude.sh

restartsleep_other=10
restartsleep_chi=5
restartsleep_qmgr=7

 function chk_service
 {
		c_rc=0
# check current
       runall=yes
       OBJCLASS=$1
       FILTER1=$2
       FILTER2=$3
       OBJECT=$4
         
       RUNIST=`echo "DISPLAY ${OBJCLASS}(${OBJECT}) where (CONTROL EQ QMGR) ALL" | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"  2>/dev/null |$GREP ${FILTER1} | grep ${FILTER2} | $GREP -v dis| sed -e 's/(/ /' -e 's/)/ /' |awk '{print $2}'`
         
       #set new vars
       OBJCLASS=$5
       FILTER1=$6
       FILTER2=$7
         
       RUNSOLL=`echo "DISPLAY ${OBJCLASS}(${OBJECT}) where (CONTROL EQ QMGR) ALL" | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"  2>/dev/null |$GREP ${FILTER1} | grep ${FILTER2} | $GREP -v dis| sed -e 's/(/ /' -e 's/)/ /' |awk '{print $2}'`
  
       for OBJECT1 in $RUNSOLL
          do
             if [ `echo $RUNIST | grep $OBJECT1 | wc -l` -ne 1 ]; then
                p_warning "${OBJECT1} process is not running"
				rc=2
             else
                p_info "${OBJECT1} process is up and running"
             fi
          done
         
       #start qmgr
       for OBJECT1 in $RUNSOLL
          do
             OBJCLASS=$5
             if [ `echo $RUNIST | grep $OBJECT1 | wc -l` -ne 1 ]; then
				line="$(echo "START ${OBJCLASS}(${OBJECT1}) " |do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null | $GREP ${OBJECT1})" 
        		p_info "${OBJECT1} process restarted, wait ${restartsleep_other} sec. before check"
                sleep ${restartsleep_other}
            	line="$(echo "DIS $1(${OBJECT1}) " | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"  2>/dev/null | $GREP ${OBJECT1} | $GREP -v "STDERR")"
    	    	OBJCLASS=$1
    			FILTER1=$2
    			FILTER2=$3
    			OBJECT=$4
    			RUNIST=`echo "DISPLAY ${OBJCLASS}(${OBJECT}) where (CONTROL EQ QMGR) ALL" | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"  2>/dev/null |$GREP ${FILTER1} | grep ${FILTER2} | grep -v dis| sed -e 's/(/ /' -e 's/)/ /' |awk '{print $2}'`
                if [ `echo $RUNIST | $GREP $OBJECT1 | wc -l` -ne 1 ]; then
                	p_error "${OBJECT1} process is not running. Automatic start failed"
#	            	call_logger CRONMON 4 "${OBJECT1} process on ${qmgr} is not running. Automatic start failed."
#new
                    call_logger CRONMON 4 ${OBJECT1}@${qmgr} "${SCRIPT_NAME}:${OBJECT1} process on ${qmgr} is not running."                   
	            	rc=3
               	else
            		p_info "${OBJECT1} process is up and running"
                fi          
             fi
          done
 }

 syntax() {
       echo "Syntax : ${MQSCRIPT}  <queue manager>|-all " 
       ende $1  
 }

 ende() {
       p_error "Date/Time:  `date`"                 
       p_error "Qmgr: ${qmgr}" 
       p_linie 
       rc=4
       exit $rc
 }
ttautostart() {
TTSTOP=`echo "DISPLAY CHS(*) CHLTYPE(MQTT)" | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null | grep -v SYSTEM | grep -B 1 "STATUS(STOPPED)" | grep CHLTYPE |  awk '{print $1}' | sed -e 's/CHANNEL(/ /' -e 's/)/ /'`
ALLTT=`echo "DISPLAY CHANNEL(*) CHLTYPE(MQTT) " | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null | grep -v DIS | grep -v SYSTEM | grep CHLTYPE | awk '{print $1}' | sed -e 's/CHANNEL(/ /' -e 's/)/ /' `

if [[ $TTSTOP ]]; then
for i in ${ALLTT}
        do
#                echo $i
                if [ `echo $TTSTOP | grep ${i} | wc -l` -eq 1 ]; then
                        x=`echo "DISPLAY CHANNEL(${i}) CHLTYPE(MQTT) " | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null | grep  "DESCR(NoRestart)"`
                        [[ -z $x ]] ||  continue
                        echo "${i} CHANNEL is not running"
			call_logger CRONMON 3 "MQTT CHANNELS On $qmgr"  "${i} is not running."
                        echo " Attempting to start ${i} channel"
                        echo "START CHANNEL(${i}) CHLTYPE(MQTT) " | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"
                        sleep 10
                        TTSTAT=`echo "DISPLAY CHS(${i}) CHLTYPE(MQTT)" | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}" 2>/dev/null | grep -v SYSTEM | grep -B 1 "STATUS(STOPPED)" | grep CHLTYPE |  awk '{print $1}' | sed -e 's/CHANNEL(/ /' -e 's/)/ /'`
                        if [[ -z "$TTSTAT" ]]; then
                                echo " MQ TT CHANNEL${i} is started "
				call_logger CRONMON 1 "MQTT CHANNELS On $qmgr"  "${i} is started."
                        else
                                echo " MQ TT CHANNEL${i} is not running. Automatic start failed"
				call_logger CRONMON 4 "MQTT CHANNELS On $qmgr"  "${i} is not running. Automatic start failed" 
								
                        fi
                fi
        done
fi
}

 function invoke {
    rc=0

    if [ -f ${MQTOOLS_ETC}/$MAINTENANCE ]; then
       p_warning "`date +'%D %T'`  start script "           
       p_warning "Maintenance mode activated - Monitoring/Checks not activ!"
       p_warning "done"
       rc=4
       return $rc
    fi

    # Queue-Manager name ?
    if [ -z "$1" ] ; then
      syntax $1
    else
       typeset -u qmgr=$1
    fi

    if [ ! -d ${MQ_LOG}/${qmgr} ] ; then
       p_error "There is no queue manager $bold${qmgr} $off created on this server !!!"
       rc=2
       return $rc
    fi

    #$MQMTOOLS/checknode $QM $QMDATA 

    QMRunning=`${MQ_HOME}/bin/dspmq -m ${qmgr} | tr -s '(|)' ' ' | $GREP -w ${qmgr} | awk '{print $4}' `

	#checking process
    case "$PLATFORM" in
      "HP-UX" | "SunOS")
          if [ "$QMRunning" != "Running" ]; then
             if [ ! -d ${MQ_DATA}/${qmgr}/queues ] ; then
               p_error "The queue manager ${qmgr} is not running on this node !!!"
               rc=4
               return $rc
             fi
          fi
          ;;
       "AIX")
          QmgrAktiv="N"
          if [ "$QMRunning" = "Running" ]; then
             QmgrAktiv="Y"
          else
             if [ ! -d ${MQ_DATA}/${qmgr}/queues ] ; then
                rc=4
                for VolGr in `lsvg`
                do
                   if [ $(lsvg -l  $VolGr | $GREP ${qmgr} | wc -l ) -ne 0 ] ; then
                      if [ $( lsvg  -o  |  $GREP $VolGr | wc -l ) -ne 1 ] ; then
                         echo "${MQSCRIPT} : The volumegroup  ${VolGr}  is not active   !!!" 
                      else
                         QmgrAktiv="Y"
                      fi
                   fi
               done
             else
                QmgrAktiv="Y"
             fi
          fi     
          if [ "${qmgr}Aktiv" = "N" ]
          then
             p_error "The queue manager ${qmgr} is not running on this node or is not on ${MQ_DATA}/${qmgr}   !!!"
             rc=4
             return $rc
          fi 
          ;;
       *) 
          ;;
    esac
    #echo $cls$off
    p_info "MQSeries  ${mqversion}           `date +'%D %T'` "
    p_info "QMGR-Name:    ${qmgr}"


    if [ $mqversion -ge 800 ] ; then
      #  MQ 8 and higher -->  MQ8_processes='amqzxma0|amqzmuc0|amqzmur0|amqrrmfa'
      ANZPROC1=4
      ANZPROC=`$PS $PSPARM|$GREP -v grep|$EGREP -e 'amqzxma0|amqzmuc0|amqzmur0|amqrrmfa' | grep -wc ${qmgr}`
   else
      #  MQ 7 --> MQ7_processes='amqzxma0|amqzmuc0|amqzmur0|amqzdmaa|amqrrmfa'
      ANZPROC1=5
      ANZPROC=`$PS $PSPARM|$GREP -v grep|$EGREP -e 'amqzxma0|amqzmuc0|amqzmur0|amqzdmaa|amqrrmfa' | grep -wc ${qmgr}`
   fi

    if [ $ANZPROC -ne $ANZPROC1 ]
    then
       p_error "There are not all MQ processes running on this server !!!"
       p_error "===>  Please check ( $ANZPROC found - $ANZPROC1 expected ) !!!"
       rc=2
    else
       p_info "OK - all necessary queue manager kernel processes are up and running"
       if [ $($PS $PSPARM | $GREP -v grep | $GREP amqpcsea | grep -wc ${qmgr} ) -ne 1 ] ; then
         p_warning "commandserver is not running !!! "
         p_info "the commandserver will restarted !! "
         rc=2
         do_as_mqm ${qmgr} "${MQ_HOME}/bin/strmqcsv ${qmgr}"
         sleep ${restartsleep_qmgr}
         if [ $($PS $PSPARM | $GREP -v $GREP | $GREP amqpcsea | $GREP -wc ${qmgr} ) -ne 1 ] ; then
             p_error "commandserver is still not running, please check !!! "
             rc=3
         fi
       else
          p_info "commandserver is up and running "
       fi

       if [ $($PS $PSPARM | $GREP -v $GREP | $GREP runmqchi | $GREP -wc ${qmgr} ) -ne 1 ] ; then
          p_warning "channelinitiator is not running !!! "
          p_info "channelinitiator will restarted !! "
          rc=2        
          do_as_mqm ${qmgr} "${MQTOOLS_HOME}/bin/wrapper ${MQ_HOME}/bin/runmqchi -q SYSTEM.CHANNEL.INITQ -m ${qmgr} >/dev/null 2>&1"
          sleep ${restartsleep_chi}
          if [ $($PS $PSPARM | $GREP -v $GREP | $GREP runmqchi | $GREP  -wc ${qmgr} ) -ne 1 ] ; then
             p_error "channelinitiator is still not running, please check !!! "
            rc=3
          fi
       else
          p_info "channelinitiator is running "
       fi

    # check listener
    OBJECT="LISTENER.*"
    OBJCLASS=LSSTATUS
    FILTER1=LISTENER
    FILTER2="STATUS(RUNNING)"
    OBJCLASS1=LISTENER
    FILTER11=LISTENER
    FILTER12="CONTROL(QMGR)"
    chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12

    # check triggermonitore
    OBJCLASS=SVSTATUS
    FILTER1=TRIGGER
    FILTER2="STATUS(RUNNING)"
    OBJECT="TRIGGER.*"
    OBJCLASS1=SERVICE
    FILTER11=TRIGGER
    FILTER12="CONTROL(QMGR)"
    chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12

    # check mqagents processes
    OBJCLASS=SVSTATUS
    FILTER1=AGENT
    FILTER2="STATUS(RUNNING)"
    OBJECT="AGENT.MQ*"
    OBJCLASS1=SERVICE
    FILTER11=AGENT
    FILTER12="CONTROL(QMGR)"
    chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12

    # check mqxr service status
    OBJCLASS=SVSTATUS
    FILTER1=MQXR
    FILTER2="STATUS(RUNNING)"
    OBJECT="SYSTEM.MQXR.*"
    OBJCLASS1=SERVICE
    FILTER11=MQXR
    FILTER12="CONTROL(QMGR)"
    chk_service $OBJCLASS $FILTER1 $FILTER2 $OBJECT $OBJCLASS1 $FILTER11 $FILTER12
    
    # check mqtt channel status
    XRSVSTAT=`echo "DISPLAY SVSTATUS("SYSTEM.MQXR.SERVICE") where (CONTROL EQ QMGR) ALL" | do_as_mqm ${qmgr} "${MQ_HOME}/bin/runmqsc ${qmgr}"  2>/dev/null |grep MQXR | grep "STATUS(RUNNING)" | grep -v dis| sed -e 's/(/ /' -e 's/)/ /' |awk '{print $2}'`
    if [[ $XRSVSTAT ]]; then 
	ttautostart 
    fi

    if [[ $(grep -ic "OCSPAuthentication" /var/mqm/qmgrs/${qmgr}/qm.ini) -eq 0 ]]; then
        p_error "Entry OCSPAuthentication=OPTIONAL missing in SSL stanza of qm.ini"
    else
        if [[ $(grep -ic '^#.*OCSPAuth' /var/mqm/qmgrs/${qmgr}/qm.ini) -eq 1 ]]; then
                p_error "In qm.ini, OCSPAuthentication entry is commented"
        else
                ocspauth=$(grep -i "OCSPAuthentication" /var/mqm/qmgrs/${qmgr}/qm.ini | awk -F "=" '{print $2}')
                if [[ ${ocspauth} != "OPTIONAL" ]]; then
                	p_error "The value of OCSPAuthentication in qm.ini is ${ocspauth}, standard value if OPTIONAL"
                else
                	p_info "OCSP property OCSPAuthentication is rightly configured in the qm.ini file"
                fi
        fi
    fi

    if [[ $(grep -ic "OCSPCheckExtensions" /var/mqm/qmgrs/${qmgr}/qm.ini) -eq 0 ]]; then
        p_error "Entry OCSPCheckExtensions=NO missing in SSL stanza of qm.ini"
    else
        if [[ $(grep -ic '^#.*OCSPCheck' /var/mqm/qmgrs/${qmgr}/qm.ini) -eq 1 ]]; then
                p_error "In qm.ini, OCSPCheckExtensions entry is commented"
        else
                ocspcheck=$(grep -i "OCSPCheckExtensions" /var/mqm/qmgrs/${qmgr}/qm.ini | awk -F "=" '{print $2}')
                if [[ ${ocspcheck} != "NO" ]]; then
                    p_error "The value of OCSPAuthentication in qm.ini is ${ocspcheck}, standard value if NO";
                else 
                	p_info "OCSP property OCSPCheckExtensions is rightly configured in the qm.ini file"
                fi
        fi
    fi    
    
 #Check queueopenkeeper

    ttt=$(cat /etc/mw/mqm/locals_$qmgr |grep -v "#" |grep QUEUEOPEN |grep "YES\|Y\|yes\|enabled" |wc -l)
    rrr=`ps -efww | tr "\t" " " | grep $qmgr |grep QueueOpener|wc -l`

    if [ ${ttt} -eq 1 -a ${rrr} -eq 0 ];then
        p_info "QUEUOPENKEEPER program is enabled and starting the program as it is not started";
        do_as_mqm ${qmgr} "${MQTOOLS_HOME}/bin/wrapper /opt/mw/mqm/bin/queueopenkeeper -m $qmgr" > /var/mw/logs/queueopenkeeper_${qmgr}.log;
        if [ $? -eq 0 ]; then p_info "QUEUOPENKEEPER is up and running"; else call_logger CRONMON 4 QueueOpenKeeper@${qmgr} "${SCRIPT_NAME}:QUEUOPENKEEPER for ${qmgr} could not be started"; fi

    elif [ ${ttt} -eq 1  -a  ${rrr} -eq 1 ];then
        p_info "QUEUOPENKEEPER tool is up and running"


    elif [ ${ttt} -eq 1  -a  ${rrr} -gt 1 ];then
        p_info "QUEUOPENKEEPER Multiple processes are running. Ensuring that  only one for $qmgr is running";
        for x in $(ps -efww| tr "\t" " " |grep QueueOpener |grep -v 'grep' |grep $qmgr| awk '{print $2}'); do kill -9 $x ; done
        p_info "QUEUOPENKEEPER program is enabled so starting it";
#        export MQ_LIB_PATH=/opt/mqm900/java/lib;/opt/mw/mqm/bin/wrapper /opt/bu/mqm/tools/bin/queueopenkeeper -m $qmgr > /tmp/queueopenkeeper_$qmgr.log;
        do_as_mqm ${qmgr} "${MQTOOLS_HOME}/bin/wrapper /opt/mw/mqm/bin/queueopenkeeper -m $qmgr" > /var/mw/logs/queueopenkeeper_${qmgr}.log
         if [ $? -eq 0 ]; then p_info "QUEUOPENKEEPER is up and running"; else call_logger CRONMON 4 QueueOpenKeeper@${qmgr} "${SCRIPT_NAME}:QUEUOPENKEEPER for ${qmgr} could not be started"; fi

    elif [ ${ttt} -eq 0  -a  ${rrr} -ge 1 ];then
        p_info " QUEUOPENKEEPER program is in disabled state,Stopping the running program ";
        for x in $(ps -efww| tr "\t" " " |grep QueueOpener |grep -v 'grep' |grep $qmgr| awk '{print $2}'); do kill -9 $x ; done


    elif [ ${ttt} -eq 0  -a  ${rrr} -eq 0 ];then
        p_info "QUEUOPENKEEPER program is in disabled state"
    else
    p_info "Tool"
    fi

# check variable LD_LIBRARY_PATH in mqinclude.sh 
   chk_variable_LD_LIBRARY_PATH



    fi
    if [ $rc -ne 0 ] ; then 
         p_info "Returncode for ${qmgr} = $rc"
    fi
    return ${rc}
 }      
  
 ################################################################################################
 # call the common routine for all queue managers
 . ${HOME}/mqinvokeforqmgrs.sh
