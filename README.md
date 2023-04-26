#!/bin/ksh
##############################################################################
#
# Script:          mqinclude.sh
#
# Usage:          
#
# Description:     add general vars and path and fkt
#
 ##############################################################################
#set -x

# TODO's
# - s bit , dann aber pruefen welche security neue prozesse und files haben
# Wenn eine Rueckgabewert ungleich 0 muss auch ein exit ungleich 0 erfolgen

. /etc/rc.status
. /opt/mw/bin/mw_include.sh

MQ_ROOT=/var/mqm
MQ_LOG=${MQ_ROOT}/log
MQ_DATA=${MQ_ROOT}/qmgrs
MQ_OLDLOG=${MQ_ROOT}/oldlogs
MQ_JMX_CONF=/etc/mw/jmxconf

# standard for non multi version. 
MQ_HOME_DEFAULT=/opt/mqm

MQTOOLS_HOME=/opt/mw/mqm
MQTOOLS_DATA=/var/mw/mqm
MQTOOLS_LOG=/var/mw/logs
MQTOOLS_ETC=/etc/mw/mqm
MQSCRIPT=`basename $0`
BACKUP_MQM=/var/mw/mqbackup
MAINTENANCE=SERVER-Maintenance

myHostname=`hostname`

#Expiring global units of work
DEF_AMQ_TRANSACTION_EXPIRY_RESCAN=15
DEF_AMQ_XA_TRANSACTION_EXPIRY=240

# set def for RTO
DEF_RTO_DEV=0
DEF_RTO_INT=0
DEF_RTO_PROD=60


GREP=grep
EGREP=egrep
WHOAMI=whoami
PS=ps
PSPARM="-ef"

MQUSER=mqm

PLATFORM="`uname`"

case ${PLATFORM} in
 'AIX')
   MQMHOME=/usr/local/mw/mqm
   MQ_HOME_DEFAULT=/usr/mqm
   export JAVA_HOME=/usr/mqm/ssl/jre
   ;;
 'HP-UX')
   MQMHOME=/usr/local/mw/mqm
   PSPARM="-efx"
   export JAVA_HOME=/opt/mqm/ssl/jre
   ;;
 'SunOS')
   MQMHOME=/usr/local/mw/mqm
   WHOAMI="/usr/ucb/whoami"
   PS=/usr/ucb/ps
   PSPARM="-auxww"
   export JAVA_HOME=/opt/mqm/ssl
   ;;
 'Linux')
   MQMHOME=/opt/mw/mqm/
# normal ps ef has only 80 char output, to less for the commands
   PSPARM="-efww"
   LINUXVERSION=$(grep VERSION /etc/S*SE-brand |awk '{print $3}')
   if [[ ${LINUXVERSION} -eq 12 ]]; then   
         lvcreateoptions="-y"
   fi
   ;;
 *)
   echo "Unsupported platform $PLATFORM"
   exit 1
   ;;
esac

#may be overridden for multi version installations
MQ_HOME=$MQ_HOME_DEFAULT

if [[ -z $MQ_HOME_LATEST ]] ; then
	# TODO check to implement inline and delete script  
    export MQ_HOME_LATEST=`${MQMHOME}/tools/get_latest_mqm_installpath.ksh`
fi 
#TODO remove all usages of old var name
export LATEST_MQM_DIR=$MQ_HOME_LATEST

# su only if root start the script
if [[ `id -u -n` != ${MQUSER} ]];then
        su_cmd_pre="su - ${MQUSER} -c "
        chown mqm:mqm  /var/mw/logs/*.log        
        su_cmd_mqm_keep_env="su ${MQUSER} -c "
else
        su_cmd_pre="ksh "
        su_cmd_mqm_keep_env="ksh "
fi

if [[ -d /etc/data/dev ]]
then
   DEVICE_DIR=/etc/data/dev
else
   DEVICE_DIR=/dev
fi

# Initialisierungen
if [[ ! -z "$TERM" ]];then
	cls=`tput clear`            # Bildschirm loeschen
	rev=`tput rev`
	ul=`tput smul`
	bold=`tput bold`            # Fettdruck einschalten
	off=`tput sgr0`
fi

# Text Konstanten
prompt() {
  echo "Eingabe ==> \c" >/dev/tty
}

p_logo() {
  sp="               "
  echo "$bold "    " $1$sp$off"
}

p_title() {
  sp="________________"
  echo "$sp$1$sp$off"
}

p_fehler() {
  sp="  "
  echo "$bold$sp$1$sp$off"
}
p_error() {
  echo "ERROR   : $1"
}
p_error_ts() {
  echo "ERROR   : $(date +%Y-%m-%d#%H:%M:%S) - $1"
}
p_info() {
  echo "INFO    : $1"
}

p_info_ts() {
  echo "INFO    : $(date +%Y-%m-%d#%H:%M:%S) - $1"
}

p_warning() {
  echo "WARNING : $1"
}
p_warning_ts() {
  echo "WARNING : $(date +%Y-%m-%d#%H:%M:%S) - $1"
}

p_line() {
  echo "_______________________________________\c"
  echo "_______________________________________$off"
}

hit() {
  if [[ "$cmd_flg" = "TRUE" ]]
  then
  	exit $rc
  fi 
 echo "$rev Press <RETURN> to continue!${off}\c"
#  read x
   read -t 600 x || exit 1
}

function die {
   rc=$?
   if [[ $rc -ne 0 ]]; then
      echo "$1. rc=$rc"
      exit $rc
   fi
}

function input {
#set -x
  text=$1
  var=$2
  default=$3
  set -A _opts $4
  optionLabel=""
  for option in ${_opts[@]}
  do
     [[ -z $optionLabel ]] && optionLabel="$option" || optionLabel="$optionLabel|$option"
  done
    
  while [ TRUE ]
  do
     label=$text
     [[ -z $optionLabel ]] || label="$label ($optionLabel)"   
     [[ -z $default ]] || label="$label [$default]"
     echo "$label : \c";
     read $var
     eval value=\$$var
     if [[ -z "$value" ]]; then
        if [[ ! -z $default ]];then
            eval "$var=\$default"
            echo "=> Use default '$default'"
            break
        fi
        continue
     fi
     
     if [[ ! -z $optionLabel ]];then
        # check
        _valid=FALSE
        for option in ${_opts[@]}
        do
           [[ "$option" = "$value" ]] && _valid=TRUE
        done
        
        if [[ "FALSE" = "$_valid" ]];then
           echo "=> Not a valid option"
           continue
        fi
     fi
     
     echo "=> Confirm value '$value' [yes]: \c"
     read confirm
     if [[ "yes" = "${confirm:-yes}" ]];then
        break;
     fi
  done
}

# Set the env based on mode (-m := qmgr or -p := path )
function mqsetenv {
   _mode=$1 
   _pathorqmgr=$2 
    
   if [[ -x ${MQ_HOME_LATEST}/bin/setmqenv ]]
   then
      . ${MQ_HOME_LATEST}/bin/setmqenv $_mode ${_pathorqmgr} -l
      MQ_HOME=`dspmqver -bf 128`
      rc=$?
      if [[ $rc -ne 0 ]]
      then
         MQ_HOME=$MQ_HOME_DEFAULT
         # Get MQ Version in format VMR from level , e.g. version 7.0.1
         mqversion=`dspmqver -bf 4| cut -d- -f1|cut -dp -f2`
      else
         mqversion=`dspmqver -bf 1024`
      fi
   fi
}

#  check for variable if version >= 710
function chk_variable_LD_LIBRARY_PATH {
if [ ! -z $mqversion ]; then
	if [ $mqversion -ge 710 ]; then
			pid=`ps -efww | grep -w ${qmgr} | grep amqzxma0 |grep -v grep| awk '{print $2}'`
			if [[ ! -z $pid ]]; then
				AMQ_LD_LIBRARY_PATH=`cat /proc/$pid/environ|tr '\0' '\n'|grep LD_LIBRARY_PATH|cut -d= -f2`
				if [[ -z $AMQ_LD_LIBRARY_PATH ]];then 
					p_error "No variable LD_LIBRARY_PATH found."
				else
				  p_info "found LD_LIBRARY_PATH=${AMQ_LD_LIBRARY_PATH}"
				fi
			else
				p_error "No running process 'amqzxma0' for qmgr: '$qmgr' found."
			fi
	fi
fi
}

#  check for variable 
function chk_variable_TRANSACTION_EXPIRY_ {

pid=`ps -efww | grep -w ${qmgr} | grep amqzxma0 |grep -v grep| awk '{print $2}'`
if [[ ! -z $pid ]]; then
	AMQ_TRANSACTION_EXPIRY_RESCAN=`cat /proc/$pid/environ|tr '\0' '\n'|grep AMQ_TRANSACTION_EXPIRY_RESCAN|cut -d= -f2`
	AMQ_XA_TRANSACTION_EXPIRY=`cat /proc/$pid/environ|tr '\0' '\n'|grep AMQ_XA_TRANSACTION_EXPIRY|cut -d= -f2`
	if [[ -z $AMQ_TRANSACTION_EXPIRY_RESCAN || -z AMQ_XA_TRANSACTION_EXPIRY ]];then 
		p_error "No TRANSACTION_EXPIRY variables found."
	else
	    p_info "Rescan Interval [minute(s)]: $(expr ${AMQ_TRANSACTION_EXPIRY_RESCAN} / 60000)"
		p_info "      XA Expiry [minute(s)]: $(expr ${AMQ_XA_TRANSACTION_EXPIRY} / 60000)"
	fi
else
	p_error "No running process 'amqzxma0' for qmgr: '$qmgr' found."
fi
}

# call a command as mqm if id=root  arg1=qmgr arg2=command
function do_as_mqm {
_qmgr=$1
_arg1="$2"
# su only if root start the script
if [[ `id -u -n` != ${MQUSER} ]];then
#       echo su - ${MQUSER} -c ". ${MQTOOLS_HOME}/bin/mqsetenv.sh $_qmgr;$_arg1"
        su - ${MQUSER} -c ". ${MQTOOLS_HOME}/bin/mqsetenv.sh $_qmgr;$_arg1"
else
        ksh "$_arg1"
fi
return $?
}

function stop_mqagents {
#Check the running Queue Managers:
#qmgrs=$(dspmq | grep -i running | sed 's/)/(/g' | awk -F "(" '{print $2}')
qmgrs=$(${MQ_HOME_LATEST}/bin/dspmq | grep -i running | sed 's/)/(/g' | awk -F "(" '{print $2}')
agentproccount=
#Stop the mqagents for the running Queue Managers
for qmgr in ${qmgrs}
do
    #echo "failed"
    #. /opt/mw/mqm/bin/mqsetenv.sh ${qmgr}
    #echo "failed1"
    INST_PATH=$(${MQ_HOME_LATEST}/bin/dspmq -o inst |grep -i $qmgr | sed 's/)/(/g'  | awk -F "(" '{print $6}')
    echo "stop service(AGENT.MQMONITOR)" | su mqm -c "${INST_PATH}/bin/runmqsc ${qmgr}"
    echo "stop service(AGENT.MQADMIN)" | su mqm -c "${INST_PATH}/bin/runmqsc ${qmgr}"
done

#Check if there are still any mqagent processes running
agentproccount=$(ps -ef|grep "/mqagent"|grep -v grep|awk '{print $2}' | wc -l)
#If there are still processes, kill the processes
if [[ ${agentproccount} -ne 0 ]]; then
    #killing mqagent process
    ps -ef|grep "/mqagent"|grep -v grep|awk '{print $2}'| xargs kill -9
else
    echo "No active mqagents process"
fi
}

function start_mqagents {
#Check the running Queue Managers:
#qmgrs=$(dspmq | grep -i running | sed 's/)/(/g' | awk -F "(" '{print $2}')
qmgrs=$(${MQ_HOME_LATEST}/bin/dspmq | grep -i running | sed 's/)/(/g' | awk -F "(" '{print $2}')
#Start the mqagents for the running Queue Managers
for qmgr in ${qmgrs}
do
    #echo "failed"
    #. /opt/mw/mqm/bin/mqsetenv.sh ${qmgr}
    INST_PATH=$(${MQ_HOME_LATEST}/bin/dspmq -o inst |grep -i $qmgr | sed 's/)/(/g'  | awk -F "(" '{print $6}')
    #echo "failed1"
    echo "start service(AGENT.MQMONITOR)" | su mqm -c "${INST_PATH}/bin/runmqsc ${qmgr}"
    echo "start service(AGENT.MQADMIN)" | su mqm -c "${INST_PATH}/bin/runmqsc ${qmgr}"
done
}
