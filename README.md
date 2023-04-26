#!/bin/sh
######################################################################
#
#  Program      : get_latest_mqm_installpath.ksh
#
#  Description  : return last mqm install path
#
#  Syntax       : get_latest_mqm_installpath.ksh 
#
#
#  Date         Action                         Author
#  -------------------------------------------------------------------
#  01.05.2014   first release                  Juan J. Sanz Martin
#  27.05.2014   adapted and tested for linux   Juan J. Sanz Martin
#
######################################################################

# set -x 

case "`uname`" in
  'AIX')
	  MQMDIR_tmp="/usr/mqm"
	  IPATHS="`lslpp -R ALL -l mqm.base.runtime 2>/dev/null | egrep -e 'INSTALL ROOT PATH' |  awk '{ print $5 }'`"
	  MQVERSION_tmp=0
	  INSTALLDIR=""
	  for path in $IPATHS
	  do 
		#echo $path
      if [[ "$path" = "$MQMDIR_tmp" ]] ; then 
         # For 7.0 path = /usr/mqm ->  bin dir in /usr/mqm/bin
         path=""
      fi
		mqver="`${path}/${MQMDIR_tmp}/bin/dspmqver -f 2 2>/dev/null | awk '{print $2}' | awk 'BEGIN {FS=\".\"} {print $1 $2}'`"       
      if [[ ! -z $mqver ]] ; then
         if [ $mqver -gt $MQVERSION_tmp ] ; then
            MQVERSION_tmp=$mqver
            INSTALLDIR="${path}/${MQMDIR_tmp}"
         fi
		fi
	  done 
	  ;;
  'HP-UX')
	  # To DO
	  # swlist -a location -a revision -l product MQSERIES 
	  ;;  
  'SunOS')
	  # pkginfo | grep -w mqm
	  # pkgparam pkgname BASEDIR
	  ;;
  'Linux')

  	  MQMDIR_tmp="/opt/mqm"
	  MQVERSION_tmp=0
	  INSTALLDIR=""
	  mqm_runtime_installed="`rpm -qa --qf \"%{NAME};%{VERSION};%{RELEASE};%{INSTPREFIXES}\n\" | grep MQSeriesRuntime`"
     for mq_inst in $mqm_runtime_installed
     do
   
      pkg_name=`echo "$mq_inst" | awk -F ';' '{print $1}'`
      pkg_version=`echo "$mq_inst" | awk -F ';'  '{print $2}' | tr -d '\.'`
      pkg_instpath=`echo "$mq_inst" | awk -F ';'  '{print $4}'`
      
      if [[ ! -z $pkg_version ]] && [[ "$pkg_instpath" != "(none)" ]]  ; then
         if [ $pkg_version -gt $MQVERSION_tmp ] ; then
            MQVERSION_tmp=$pkg_version
            INSTALLDIR="${pkg_instpath}/"
         fi
		fi
     
     done
     ;;
   *)
	  echo "wrong  OS System !!!"
	  # exit ?
	  ;;
esac



if [ -d $INSTALLDIR ] && [ ! -z $INSTALLDIR ] ; then
	echo $INSTALLDIR
	exit 0
else
	exit 1
fi
