
#!/bin/ksh
##############################################################################
#$Id: 
#
# Script:           mqserver-check.sh
#
# Usage:          
#
# Description:      check the ILSE mq installation
#                     
#
# Remarks:         
#                  
#                  
#
# History:         2012-07-25   lreckru     initial version
#                  2012-07-30   lreckru     add check for I_mqserver_cg_unix
#                  2012-08-10   lreckru     add check for TSM, HPOV, commandline parameter
#                  2012-08-31   lreckru     add check for repository status (must be disabled)
#                  2012-08-31   lreckru     change check for repository status to "CGMQ"
#                  2012-08-31   lreckru     change TSM Check - no piping to /dev/null
#                  2013-02-13   lreckru     change zypper -n refresh >/dev/null only erroroutput
#                  2013-03-20   lreckru     new word option for grep (-w) and ${repo_pattern}
#                  2013-08-28   lreckru     new backup check structure
#				   2014-01-15   lreckru     zypper  refresh to zypper --non-interactive refresh
#                  2014-01-20   lreckru     better change to --gpg-auto-import-keys  
#                                           Automatically trust and import new repository signing keys.
#                  2014-02-24   lreckru     remove check of $I_ilmt_agent_config
#                  2014-04-24   lreckru     add NetBackup 
#                  2014-04-30   lreckru     add LEGATO Work test 
#                  2015-03-31	jduelli		changed backup test to show all available backup systems 
#                  2015-08-07   lreckru     add return value, add sum of checks output at end 
#                  2015-10-09   lreckru     add ocsp provider check
#                  2016-03-18   lreckru     Check installed MQSeries packages
#                  2016-03-18   lreckru     redesign "Check installed tools packages"
#                  2016-03-21   lreckru     redesign output Check installed  packages
##############################################################################
#
#set -x
#
# get full path to this script

PROG=`readlink -f $0`
TOOL_HOME=$PROG
TOOL_HOME=`dirname $TOOL_HOME`

MQTOOLS_HOME=/opt/mw/mqm

# call the common include
. ${MQTOOLS_HOME}/bin/mqinclude.sh
. ${TOOL_HOME}/mqserver-include.sh
. ${TOOL_HOME}/../config/check.conf
. ${TOOL_HOME}/includeMQ.sh

DATE=`date +"%Y-%m-%d#%H-%M"`
NAME=`echo $(uname -n) $(uname -s) $(uname -r) $(uname -m)`
# repo_pattern="CGMQ"
erg_1=0

linuxversion=$(grep VERSION /etc/S*SE-brand |awk '{print $3}')

# check if ILMT should be installed
#if [[ -f ${ILMT_check_file} ]]
#then
# no check 
#        no_ILMT_check=TRUE
#else
#        no_ILMT_check=FALSE
#fi


# scan for default backupsystem
_BackUp_def="n/a"
if [[ -f ${backup_file} ]]
then
	for w in `cat ${backup_file}`;do
        case $w in
        lgtoclnt|lgtoman)
                _BackUp_def="EMC NetWorker"
                 ;;
        TIVsm-API64|TIVsm-BA)
                _BackUp_def="TSM (Tivoli Storage Manager)"
                 ;;
        SYMCpddea|SYMCnbjre|SYMCnbjava|SYMCnbclt)
                _BackUp_def="Netbackup"
                 ;;
        *) ;;
       esac
	done
fi

##############################################################
function check_package {
	_pkg_name=$1
	VERSION_exp=
	VERSION_inst=
	_erg_chk=0
	
	for u in $(zypper se -s ${_pkg_name}|grep " ${_pkg_name} "|grep -v '(System Packages)'|cut -d'|' -f4|sed 's/-/./g');do
		    VERSION_exp=$u
			if [[ ! "$?" -eq "0" ]];then 
				VERSION_inst="n/a"
			fi
		    break
	done
        for u in $(zypper se -s ${_pkg_name}|grep " ${_pkg_name} "|grep "i |\|i+ |"|cut -d'|' -f4|sed 's/-/./g');do
			VERSION_inst=$u
			if [[ ! "$?" -eq "0" ]];then
				 VERSION_inst="n/a"
			fi
			break
	done
	VERSION_exp=${VERSION_exp:-"n/a"}
	VERSION_inst=${VERSION_inst:-"n/a"}
	if [[ "${VERSION_exp}" == "${VERSION_inst}" ]];then
			if [[ "${VERSION_exp}" == "n/a" ]];then
	    		status="not installed / no version available"
				_erg_chk=1		
			else
	    		status="up to date"
			fi
	else
			if [[ "${VERSION_exp}" == "n/a" || "${VERSION_inst}" == "n/a" ]];then
					if [[ "${VERSION_inst}" == "n/a" ]];then
						 status="not installed / newer version available"
						_erg_chk=1
					else
					   status="installed / no version from repo available"
					fi
			else
					VERSION_big=$(echo "${VERSION_exp}\n${VERSION_inst}"|sort -Vr|head -n1)
					if [[ "${VERSION_inst}" == "${VERSION_big}" ]];then
						status="newer version installed / older version available"
					else
					  status="installed / newer version from repo available"
					  _erg_chk=1
					fi
			fi
	fi
	if [[ ${_erg_chk} == 1 ]];then
   				p_error "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" $_pkg_name ${VERSION_inst} ${VERSION_exp} "${status}")"
				erg_1=$_erg_chk
	else
   				p_info "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" $_pkg_name ${VERSION_inst} ${VERSION_exp} "${status}")"
	fi
}
##############################################################

# checks 
echo =================================================
echo "Check the MQ installation on $DATE" 
echo "with $PROG"
# location
echo "Location: $NAME"
echo =================================================
echo "           Check MQ-Version(s)                   "
echo "-------------------------------------------------"
if [[ -f "/opt/mqm${MQ_VERSION_LATEST}/bin/dspmqver" ]];then 
	/opt/mqm${MQ_VERSION_LATEST}/bin/dspmqver -i
else
	dspmqver
fi
    
QMCount=$(${MQ_HOME_LATEST}/bin/dspmq -o installation | wc -l)
OCSPValueYES=$(grep -i "OCSPCheckExtensions=YES" /var/mqm/qmgrs/*/qm.ini | wc -l)
OCSPValue=$(grep -i "OCSPCheckExtensions" /var/mqm/qmgrs/*/qm.ini | wc -l)

if [[ $OCSPValueYES -ne 0 || $OCSPValue -lt $QMCount ]]; then  
    
    echo =================================================
    echo "           Check OCSP provider                   "
    echo "-------------------------------------------------"
    # check for reach all ocsp provider (needed for SSL)

    ok_count=0
    g_count=$(echo "${ocsp_svr_lst}"| wc -w)

    if [[ "$g_count" -gt 0 ]]
    then
        for name in ${ocsp_svr_lst};do
        	echo "        check ${name}"           
            ${MQTOOLS_HOME}/bin/tcpconnecttest -p 80 -t ${name} > /dev/null 2>&1
				if [[ "$?" -eq "0" ]];then 
					ok_count=$(expr $ok_count + 1)
				fi	
	    done
		if [[ "$g_count" -eq "$ok_count" ]]
		then
		    p_info "OCSP provider($ok_count) reached"
		else
		    p_error "$(expr $g_count - $ok_count) of $g_count OCSP provider not reached"
	    fi
    else
        p_error "OCSP config not found"
    fi

else
	echo =================================================
    p_info " Skip 'Check OCSP provider' by config of qm.ini"
fi

#if [[ ${no_ILMT_check} == "FALSE" ]]
#then
#        echo =================================================
#        echo "           Check ILMT configuration              "
#        echo "-------------------------------------------------"
# check for variables 'scan_group =' and 'server ='
#        echo check the /etc/tlmagent.ini
#        if [[ -f /etc/tlmagent.ini ]];then
#                grep "[e|u][r|p] = " /etc/tlmagent.ini
#        else
#            p_warning "file not found /etc/tlmagent.ini"
#            erg_1=1
#        fi
#else
#        echo =================================================
#        p_info "Skip 'check ILMT' by config of ${ILMT_check_file}"
#fi

if [ "${linuxversion}" -ge "12" ]; then
        FlexInstStatus=`systemctl status ndtask | grep "Loaded:" | grep "not-found" | wc -l`
		if [[ ${FlexInstStatus} -ne 1 ]]; then
                flexrunst=`systemctl status ndtask | grep Active | sed 's/)/(/g' | awk -F "(" '{print $2}'`

                if [[ "${flexrunst}" == "running" ]]; then
                        p_info "Status of Flexera:- ${flexrunst}"
                else
                        p_error "Status of Flexera:- ${flexrunst}"
                        erg_1=1
                fi
        else
                p_error "Flexera is not installed"
        fi
fi


echo =================================================
echo "           Check directory                       "
echo "-------------------------------------------------"
#check for backup directory
if [[ -d $I_backupname ]]
then 
    p_info "BackUpDirectory '$I_backupname' exist" 
else
    p_error "BackUpDirectory '$I_backupname' not found"
    erg_1=1
fi

if [ "${linuxversion}" -ge "12" ]; then 
    if [[ -d $I_newbackupname ]]
    then 
        p_info "NewBackUpDirectory '$I_newbackupname' exist" 
    else
        p_error "NewBackUpDirectory '$I_newbackupname' not found"
    erg_1=1
    fi
fi

echo =================================================
echo "           Check Volume Group                    "
echo "-------------------------------------------------"
# Sample output:
# PV /dev/sdd    VG data2vg   lvm2 [234.00 GiB / 234.00 GiB free]
# PV /dev/sdc    VG data1vg   lvm2 [101.00 GiB / 75.81 GiB free]
# PV /dev/sdb1   VG vgswap    lvm2 [3.97 GiB / 64.00 MiB free]
# PV /dev/sda2   VG vg00      lvm2 [19.72 GiB / 3.25 GiB free]
# Total: 4 [358.68 GiB] / in use: 4 [358.68 GiB] / in no VG: 0 [0   ]
# important 

ok=FALSE
a=$(pvscan 2>/dev/null | grep "sdd " | grep "data2vg"| wc -l)
if [[ "$a" -eq "1" ]]
then
      ok=TRUE    
else
	p_error "data2vg   must be on   /dev/sdd"
    erg_1=1
fi

a=$(pvscan 2>/dev/null  | grep "sdc " | grep "data1vg"| wc -l)
if [[ "$a" -eq "1" ]]
then
     ok=TRUE
else
	p_error "data1vg   must be on   /dev/sdc"
    erg_1=1
fi

if [ "${linuxversion}" -ge "12" ]; then
    a=$(pvscan 2>/dev/null  | grep "sde " | grep "backupvg"| wc -l)
    if [[ "$a" -eq "1" ]]
    then
       ok=TRUE
    else
        p_error "backupvg   must be on   /dev/sde"
        erg_1=1
    fi
fi


echo
pvscan
 
echo =================================================
echo "           Check File System                     "
echo "-------------------------------------------------"

if [ "${linuxversion}" -ge "12" ]; then
       # logvols="/dev/data1vg/lvvar_itlm /dev/data1vg/lvvar_mqm_errors /dev/backupvg/lvvar_mw_qbackup /dev/backupvg/lvvar_mw_backup"
		logvols="/dev/data1vg/lvvar_itlm /dev/data1vg/lvvar_mqm_errors /dev/backupvg/lvvar_mw_qbackup /dev/backupvg/lvvar_mw_mqbackup /dev/backupvg/lvvar_mw_backup"
fi

if [ "${linuxversion}" -lt "12" ]; then
        logvols="/dev/vg00/ilmt_lv /dev/data1vg/lvvar_mqm_errors /dev/data1vg/lvvar_mw_mqbackup"
fi

for logvol in ${logvols};
do
        error=$(lvdisplay ${logvol} 2>&1)
        rc=$?
        if [ $rc -ne 0 ]; then
                if [[ "${logvol}" != "/dev/data1vg/lvvar_itlm" && "${logvol}" != "/dev/vg00/ilmt_lv" ]]; then
#                        p_error "FS check failed for '${logvol}' '${error}'"
#                       p_error "MQ Errors missing: '${error}'"
#                        erg_1=1
				###new
					if [[ "${logvol}" == "/dev/backupvg/lvvar_mw_qbackup" ]]; then
						error1=$(lvdisplay /dev/backupvg/lvvar_mw_mqbackup 2>&1)
						if [ $? -ne 0 ]; then	
							p_error "FS check failed for '${logvol}' '${error}'"
							erg_1=1
						fi
					elif [[ "${logvol}" == "/dev/backupvg/lvvar_mw_mqbackup" ]]; then
						error1=$(lvdisplay /dev/backupvg/lvvar_mw_qbackup 2>&1)
						if [ $? -ne 0 ]; then	
							p_error "FS check failed for '${logvol}' '${error}'"
							erg_1=1
						fi					
					else
                p_error "FS check failed for '${logvol}' '${error}'"
#               p_error "MQ Errors missing: '${error}'"
                erg_1=1
                fi
				###	
                fi
        else
                if [[ "${logvol}" == "/dev/data1vg/lvvar_itlm" || "${logvol}" == "/dev/vg00/ilmt_lv" ]]; then
                        p_warning "FS ${logvol} found, needs to be removed"
        else
                p_info "FS check successful for '${logvol}'"
        fi
        fi
done

echo ==================================================
echo "          Check Qmgr Filesystem Type            "
QMGRS=$(${MQ_HOME_LATEST}/bin/dspmq -o inst |awk -F " " '{print $1}' | sed 's/)/(/g' | awk -F "(" '{print $2}')
for qmgr in ${QMGRS}
do
        echo ------------------------------------------------
        echo "Filesystem Type for Queue Manager: $qmgr"
        FSTYPE_DATA=`df -hT | grep ${qmgr} | grep "qmgrs/" | awk -F " " '{print $2}'`
        FSTYPE_LOG=`df -hT | grep ${qmgr} | grep "log/" | awk -F " " '{print $2}'`
        FSTYPE_OLDLOG=`df -hT | grep ${qmgr} | grep "oldlogs/" | awk -F " " '{print $2}'`
        #echo "Data:    ${FSTYPE_DATA}"
                if [ "${linuxversion}" -ge "12" ]; then
        if [[ "${FSTYPE_DATA}" == "xfs" ]]; then
                p_info "Data:    ${FSTYPE_DATA}"
        else
                p_error "Data:    ${FSTYPE_DATA}"
                erg_1=1
        fi
        if [[ "${FSTYPE_LOG}" == "xfs" ]]; then
                p_info "Log:     ${FSTYPE_LOG}"
        else
                p_error "Log:     ${FSTYPE_LOG}"
                erg_1=1
        fi
        if [[ "${FSTYPE_OLDLOG}" == "xfs" ]]; then
                p_info "OldLogs: ${FSTYPE_OLDLOG}"
        else
                p_error "OldLogs: ${FSTYPE_OLDLOG}"
                erg_1=1
        fi
                fi
                if [ "${linuxversion}" -lt "12" ]; then
                        if [[ "${FSTYPE_DATA}" == "ext3" ]]; then
                p_info "Data:    ${FSTYPE_DATA}"
                        else
                p_error "Data:    ${FSTYPE_DATA}"
                erg_1=1
                        fi
                        if [[ "${FSTYPE_LOG}" == "ext3" ]]; then
                p_info "Log:     ${FSTYPE_LOG}"
                        else
                p_error "Log:     ${FSTYPE_LOG}"
                erg_1=1
                        fi
                        if [[ "${FSTYPE_OLDLOG}" == "ext3" ]]; then
                p_info "OldLogs: ${FSTYPE_OLDLOG}"
                        else
                p_error "OldLogs: ${FSTYPE_OLDLOG}"
                erg_1=1
                        fi
                fi
done

#
echo =================================================
echo "           Check installed tools packages        "
echo "-------------------------------------------------"
for pkg_name in $I_mqserver_cg_unix $I_mw_base $I_cgmq_wmq $I_mqtools_cg_unix;do
	pkg_name=${pkg_name%=*}
	check_package ${pkg_name}
done
#pkg_name=$I_ILMT_TAD4D_agent
#pkg_name=${pkg_name%=*}
#if [[ ! -z ${no_ILMT_check} ]]
#        then
#                check_package ${pkg_name}
#else
#                p_info "Skip 'check ${pkg_name}' by config of ${ILMT_check_file}"
#fi
echo "-------------------------------------------------"
echo "           Check installed MQSeries packages     "
echo "-------------------------------------------------"

for mqver in `zypper se -s "MQSeriesRuntime_" | grep -v "\-U" | grep -v '(System Packages)'|grep "| package" | awk -F "|" '{print $2}' | sort -ru`; do
mqvervalue=$(echo ${mqver} | awk -F "_" '{print $2}')
#for u in `zypper se -s MQSeriesRuntime_|grep MQSeriesRuntime|grep -v '(System Packages)'|cut -d'|' -f4|sed 's/-/./g'|sort -r`;do
for u in `zypper se -s ${mqver}|grep MQSeriesRuntime|grep -v '(System Packages)'|cut -d'|' -f4|sed 's/-/./g'|sort -r`;do
	MQ_VERSION=$u
        if [[ -f /etc/bu/mw_noupdatemq ]]; then
                . "/etc/bu/mw_noupdatemq"
                eval fixedmqversion=\${$mqvervalue}
                if [[ "$fixedmqversion" != "" ]]; then
                        MQ_VERSION=$fixedmqversion
                fi
        else
                fixedmqversion=""
        fi
	break
done

#for u in `zypper se -s MQSeriesRuntime_|grep MQSeriesRuntime|grep 'i |'|cut -d'|' -f4|sed 's/-/./g'|sort -r`;do
#for u in `zypper se -s ${mqver}|grep MQSeriesRuntime|grep 'i |'|cut -d'|' -f4|sed 's/-/./g'|sort -r`;do
for u in `zypper se -s ${mqver}|grep MQSeriesRuntime|grep "i |\|i+ |"|cut -d'|' -f4|sed 's/-/./g'|sort -r`;do
	MQ_VERSION_inst=$u
	break
done
if [[ -z ${MQ_VERSION} && -z ${MQ_VERSION_inst} ]];then 
		erg_1=1
		p_error "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" MQSeriesRuntime n/a n/a 'not installed /cannot get the top version from repository')"
else
		if [[ -z ${MQ_VERSION} ]];then 
			erg_1=1
			p_error "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" MQSeriesRuntime ${MQ_VERSION_inst} n/a 'installed / cannot get the top version from repository')"
		else
                        msg=$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" MQSeriesRuntime ${MQ_VERSION_inst} ${MQ_VERSION} 'installed / ')
                        if [[ "$fixedmqversion" == "" ]]; then
                                msg=$msg"highest version from repository"
                        else
                                msg=$msg"fixed version from file"
                        fi
                        if [[ "${MQ_VERSION}" == "${MQ_VERSION_inst}" ]]; then
                                p_info "$msg"
                        else
                                if [[ -z "${MQ_VERSION_inst}" ]]; then
									p_info "$msg"
								else
                                p_error "$msg"
                                erg_1=1
		fi
fi
fi
fi

#MQ_PKG=$(zypper se -s MQSeries*|grep MQSeries | grep 'i |'|cut -d'|' -f2|cut -d'-' -f1|cut -d'_' -f1|sort -ur)
MQ_PKG=$(zypper se -s MQSeries*|grep MQSeries | grep "i |\|i+ |" |cut -d'|' -f2|cut -d'-' -f1|cut -d'_' -f1|sort -ur)
for PKG in ${MQ_PKG}
do
#zypper se -s $PKG*|grep $PKG|grep 'i |'|sort -ur | while read line
zypper se -s $PKG*${mqvervalue}*|grep $PKG|grep "i |\|i+ |"|sort -ur | while read line
   do
	if [[ "$PKG" == "MQSeriesRuntime" ]];then continue;fi
    MQ_VERSION_akt=$(echo $line |cut -d'|' -f4|sed 's/-/./g'|sed 's/ //g')
	if [[ "${MQ_VERSION_akt}" == "${MQ_VERSION_inst}" ]];then
		p_info "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" $PKG ${MQ_VERSION_akt} ${MQ_VERSION_inst} 'up to date / installed MQSeriesRuntime Version')"
	else
    	p_error "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" $PKG ${MQ_VERSION_akt} ${MQ_VERSION_inst} 'different to installed MQSeriesRuntime Version')"
    	erg_1=1
    fi
    break
   done
done
done

#################################

mqverrepo=$(zypper se -s "MQSeriesRuntime_" | grep -v "\-U" | grep -v '(System Packages)'|grep "| package" | awk -F "|" '{print $2}' | awk -F "_" '{print $2}' | sort -ru)
#mqverrepocount=$(echo ${mqverrepo} | wc -l)
mqverrepocount=$(zypper se -s "MQSeriesRuntime_" | grep -v "\-U" | grep -v '(System Packages)'|grep "| package" | awk -F "|" '{print $2}' | awk -F "_" '{print $2}' | sort -ru | wc -l)
mqverrepo=$(echo "${mqverrepo}" | tr "\n" " " | tr -s " ")

mqverinst=$(${MQ_HOME_LATEST}/bin/dspmqinst | grep -i "InstPath:" | awk -F "/" '{print $3}' | sort -r)
#mqverinstcount=$(echo ${mqverinst} | wc -l)
mqverinstcount=$(${MQ_HOME_LATEST}/bin/dspmqinst | grep -i "InstPath:" | awk -F "/" '{print $3}' | sort -r | wc -l)
mqverinst=$(echo "${mqverinst}" | tr "\n" " " | tr -s " ")

p_info "MQ installations available from zypper repository search: ${mqverrepo}"
p_info "MQ installations locally installed on the Server:         ${mqverinst}"

if [ ${mqverrepocount} -ne ${mqverinstcount} ]; then
        p_warning "Mismatch found in MQ version in suse channels and installed on the server"
        #erg_1=1
fi


#################################


# for prog_0 in ${MQ_PKG_800};do
# 	found_prog=0
# 	for prog_1 in ${MQ_PKG};do
# 		if [[ "${prog_0}" == "${prog_1}" ]];then
# 			found_prog=1
# 			break
# 		fi
# 	done
# 		if [[ "${found_prog}" -ne "1" ]];then
# 			p_error "$(printf "%-18s Version(Cur/Exp) %12s / %-12s %s" ${prog_0} n/a ${MQ_VERSION_inst} 'not installed / installed MQSeriesRuntime Version')" 
# 			erg_1=1
# 		fi
# done

##########################################################
#                     Backup
_NetBackUp=NOTOK
_NetBackUp_WORK=NOTOK
_LEGATO=NOTOK
_LEGATO_WORK=NOTOK
_TSM=NOTOK
_TSM_WORK=NOTOK
suchstr="nsrexecd"
erg_tsm=1
_erg_summary_work=0
_erg_summary=0

# check EMC NetWorker
erg_legato=$(ps -efww | grep ${suchstr} | grep -v grep | wc -l)
if [[ "${erg_legato}" -gt "0" ]]
then
	_LEGATO=OK
    _erg_summary=$(expr $_erg_summary + 1)
    erg_legato2=`echo versions /var/mqm/mqs.ini | recover 2>&1 | grep mqs.ini | wc -l`
    if [[ "${erg_legato2}" -gt "1" ]]
    then
          _LEGATO_WORK=OK
	      _erg_summary_work=$(expr $_erg_summary_work + 1)
    fi
fi

#Check TSM
#check if file exist
which dsmc  > /dev/null 2>&1
if [[ "$?" == "0" ]]
then 
	if [[ -f $(which dsmc) ]] ;then 
		_TSM=OK
		_erg_summary=$(expr $_erg_summary + 1)
		# Querying what partitions have been backed up with TSM
		dsmc q fi > /dev/null 2>&1
		erg_tsm=$?
  		  	case ${erg_tsm} in
        		0|8) _TSM_WORK=OK
            		_erg_summary_work=$(expr $_erg_summary_work + 1)
            		;;
			esac  
	fi
fi
# Tivoli Storage Manager backups can complete successfully 
# even if the client receives errors or warnings. 
# When running a backup using a script, there is no ability to output the errors 
# or warnings that would normally be seen in the client error log (dsmerror.log).
# Process Exit Code 8 translates to Tivoli Storage Manager return code 8. 
# This means the backup completed, but witnessed one or more warnings.#

#Check Netbackup
if [[ -f /usr/openv/netbackup/bin/bplist ]];then
	_NetBackUp=OK
	_erg_summary=$(expr $_erg_summary + 1)
 	erg_NetBackUp=`/usr/openv/netbackup/bin/bplist -l /var/mqm/mqs.ini | wc -l`
    if [[ "${erg_NetBackUp}" -gt "1" ]]
    then
            _NetBackUp_WORK=OK
    		_erg_summary_work=$(expr $_erg_summary_work + 1)
    fi
fi

# output for all
echo "================================================="
echo "           Check Backup system (Summary)         "
echo "-------------------------------------------------"
p_info "default Backup system is '${_BackUp_def}'"

if [[ "$_erg_summary_work" == "0" ]];then
    p_info "no working backup program found"
    #erg_1=1
else
	p_info "${_erg_summary_work} of ${_erg_summary} Recovery Tests successful"
fi

# print results
if [[ "$_NetBackUp" == "OK" ]];then
    if [[ "$_NetBackUp_WORK" == "OK" ]];then
    	echo "              Netbackup Client Test OK "
    else
        echo "              Netbackup Client Test (no backup of mqs.ini found)"
    fi
fi
if [[ "$_LEGATO" == "OK" ]];then
	    if [[ "$_LEGATO_WORK" == "OK" ]];then
	        echo "              EMC NetWorker Recover Test OK"
	    else
	        echo "              EMC NetWorker Recover Test failed"
	    fi
fi

if [[ "$_TSM" == "OK" ]];then
  	case ${erg_tsm} in
        0|8) echo "              TSM Recover Test OK"
             ;;
		12)  echo "              TSM not completely configured."
		   	;;
		*)   echo "              TSM number: $erg_tsm"
		    ;;
	  esac  
fi

#########################################################
if [[ -f /opt/OV/bin/ovcd ]]; then
echo =================================================
echo "           Check HP OpenView                     "
echo "-------------------------------------------------"
#
suchstr="ovcd"
erg=$(ps -efww | grep ${suchstr} | grep -v grep | wc -l)
if [[ "$erg" -gt "0" ]]
then
    ps -ejH | grep ${suchstr}
    echo ""
	p_info "HPOpenView main process '${suchstr}' found"
else
	p_error "HPOpenView main process '${suchstr}' not found"
    erg_1=1
fi
fi

if [[ -d "/opt/OV/rest_ws" ]]; then
echo "-------------------------------------------------"
        p_info  "Lightweight OVO monitoring is in place"
echo "-------------------------------------------------"
fi

#
# echo =================================================
# echo "           Check/Update Repository CGMQ          "
# echo "-------------------------------------------------"
# # only refresh CGMQ Repo
# ${TOOL_HOME}/mq_repository.sh -on >/dev/null || die "ERROR Activate repository"
# for repo_pattern1 in $(zypper ls | grep  ${repo_pattern} | cut -d "|" -f3  | cut -d " " -f2)
# 	do 
# 	   zypper --gpg-auto-import-keys -n refresh ${repo_pattern1} >/dev/null
# 	done
# ${TOOL_HOME}/mq_repository.sh -off >/dev/null
# 
# repo_name=$(zypper ls | grep -w ${repo_pattern} | cut -d "|" -f3  | cut -d " " -f2)
# repo_state=$(zypper ls | grep -w ${repo_pattern} | cut -d "|" -f4  | cut -d " " -f2)
# zypper ls
# erg=$?
# if [[ "$erg" -ne "0" ]];then erg_1=$erg; fi
# echo ""
# if [[ "${repo_state}" == "Yes" ]];then
#   p_error "Repository ${repo_name} enabled"
#   erg_1=1
#   echo "         use the script '/opt/mw/install/mq_repository.sh -off'"
# else
#   p_info "Repository ${repo_name} disabled"
# fi


echo =================================================
if [[ "${erg_1}" -gt "0" ]]
then
	echo "        END CHECK with ERRORS                   "
else
	echo "             END CHECK - OK                     "
fi
echo =================================================
return ${erg_1}
