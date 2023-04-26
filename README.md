#!/bin/ksh
##############################################################################
#$Id: mqqueueaut.sh,v 1.1 2012/02/16 15:17:51 emea\fdressle Exp $
#
# Script:         mqqueueaut.sh
#
# Usage:          mqqueueaut.sh -qmgr <qmgrname> -queue <queuename> [-group <groupname>] [-hop]
#
# Description:    applies put authorities for application and/or channel
#                  
#                     
#
# Remarks:         
#                  
#                  
#                  
#                  
#
# History:        2012/02/16  fdressle
#                 2014/11/25  sanzmaj  multiversion support
#
##############################################################################
#set -x


# get full path to this script
HOME=`readlink -f $0`
HOME=`dirname $HOME`

# call the common include
. ${HOME}/mqinclude.sh

function usage {
   echo "Usage: $MQSCRIPT -qmgr <qmgrname> -queue <queuename> [-group <groupname>] [-hop]"
   echo ""
   echo "qmgrname:   Name of queue manager."
   echo "queuename:  Name of queue for which oam is applied."
   echo "groupname:  Name of application group used for put authorization. Optional."
   echo "hop:        Remote queue is used to forward messages. Grant receivers. Optional."
   exit 1
}

if [ $# -eq 0 ]; then
   usage
fi

while [ $# -ne 0 ]; do
   case $1 in
      # lower case
      -group)
         typeset -l ${1#-}="$2";shift;;
	     		
      # upper case
      -qmgr|-queue)
         typeset -u ${1#-}="$2";shift;;
	     		
      # single
      -hop)
         typeset ${1#-}="true";;
      -*) echo "Unkown option $1";exit 2;;
      *)  ;;
   esac
   shift
done

if [[ -z "${qmgr}" || -z "${queue}" ]]
then
   echo "Missing required options."
   exit 1
fi


# Checks and set multi-version vars
mqsetenv -m $qmgr


# TODO check if queue contains wildcards

if [[ ! -z "${group}" ]]
then
   ${su_cmd_pre} "${MQ_HOME}/bin/setmqaut -m ${qmgr} -t q -n \"${queue}\" -g \"${group}\" -all +put +inq"
   ${su_cmd_pre} "${MQ_HOME}/bin/dspmqaut -m ${qmgr} -t q -n \"${queue}\" -g \"${group}\""
fi

if [[ ! -z "${hop}" ]]
then
   ${su_cmd_pre} "${MQ_HOME}/bin/setmqaut -m ${qmgr} -t q -n "${queue}" -g mcadlq -n \"${queue}\" -all +put +setall"
   ${su_cmd_pre} "${MQ_HOME}/bin/dspmqaut -m ${qmgr} -t q -n "${queue}" -g mcadlq"
   ${su_cmd_pre} "${MQ_HOME}/bin/setmqaut -m ${qmgr} -t q -n "${queue}" -g mcauser -all +put +setall"
   ${su_cmd_pre} "${MQ_HOME}/bin/dspmqaut -m ${qmgr} -t q -n "${queue}" -g mcauser"
fi
