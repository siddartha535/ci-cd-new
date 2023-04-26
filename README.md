
#!/usr/bin/ksh
 #
 #       mqsslsetup.sh: Anlegen und Handling von SSL Zertifikaten eines Queue Managers 
 #
 #                              Author: Juergen Drescher (04.2007)
 #
 #                
 #
 #        Die Voraussetzung ist das ein aktuelles GSKit auf dem System installiert ist
 #
 #   Call syntax: sslsetup <Queue Manager> [option]
 #
 # History:         
 #       
 #       2013-01-15   lreckru     add menu point Activate Duplicated KeyDB
 #       2013-01-25   lreckru     Version number as variable 1.6.0
 #       2013-07-15   lreckru     chown mqm:mqm after create 
 #       2014-08-18   lreckru     add get expiry date, cmd line version 
 #	     2014-11-27	  sanzmaj	    multiversion
 #					  phutzel	    checks
 #       Version 1.8.0			  
 #       2015-03-11   fdressle,    Select SSL MQ Command depending of MQ Version
 #                    lreckru      option d) changed output for Version 8
 #       Version 1.8.1			  
 #       2015-03-31   lreckru	   add cmd line option 3,6,13
 #       2015-04-15   lreckru	   add cmd line option f,d
 #                                 (option 3) - look for certificates in $QMDATA/$SSLDIR/,/opt/bu/mqm/tools/cert, /opt/mw/mqm/cert
 #       2015-04-15   lreckru	   add cmd line option g, replace certificate
 #								   add output of CN for option d
 #       2015-05-13   lreckru      add switch norefssl for cmd line option g
 #       2015-08-04   mishrbi      removed "-format binary" in SSLCOMMAND from (option 3)
 #		 2016-01-29   lreckru      add cmd line option 14 (Change 31805502 )
 #		 2016-02-24   lreckru      add cmd line option 16 (show key db pwd expiry )
 ###############################################################################

SCRIPTVERSION=1.8.4

 # get full path to this script
 HOME=`readlink -f $0`
 HOME=`dirname $HOME`
 cmd_flg=FALSE
 ssl_exp_filename=/tmp/ssl_list
 
indir_1=/opt/bu/mqm/tools/cert
indir_2=/opt/mw/mqm/cert


 # call the common include
 . ${HOME}/mqinclude.sh

 SSL_CONF=${MQTOOLS_HOME}/config/mqsslsetup.cfg

 # load configuration
 if  ${su_cmd_pre} "[ -f $SSL_CONF ]"; then
    . $SSL_CONF
 else
    echo "Error: Config file mqsslsetup.cfg not found or not executable!"
    exit
 fi

 syntax() {
   echo "Syntax : mqsslsetup [Queue-Manager] [commandnumber [parameter]]" 
   echo "commandnumber:\n  d    Show Certificate expire date"
   echo "commandnumber:\n  f    Remove all expired Certificate"
   echo "commandnumber:\n  g    Replace Certificate; <CN of certificate> <FQN of certificate> <new Label of certificate> [norefssl] no ssl refesh cmd"
   echo "commandnumber:\n  3    Import Root CA; <Label of certificate> <filename of certificate (to find in $QMDATA/$SSLDIR)>"
   echo "commandnumber:\n  6    Import signed Certificate; parameter: filename of certificate (to find in $QMDATA/$SSLDIR)"
   echo "commandnumber:\n  13   Remove Certificate; parameter: Label of certificate"
   echo "commandnumber:\n  14   Change KeyDB Password incl. Expiry; parameter: pwd expiry in days. ! new pwd input required ! -"
   echo "       use this only with remote_sslsetup.sh"
   echo "       or use somthing like 'echo pwdnew |lssh xboi165e \"read x;echo \\\$x\" '"
   echo "commandnumber:\n  16   Show KeyDB Password Expiry; pwd input required ! -"
   echo "       use this only with remote_sslsetup.sh"
   echo "       or use somthing like 'echo pwd |lssh xboi165e \"read x;echo \\\$x\" '"
   exit $1
 }

#
refresh_security(){
	echo "REFRESH SECURITY TYPE (SSL)" | ${su_cmd_pre} ". ${MQTOOLS_HOME}/bin/mqsetenv.sh $qmgr;${MQ_HOME}/bin/runmqsc ${qmgr}"
}

# get certname from label
get_cert_name(){
_cert_info=""
_cert_name=""
_cert_label=""
_no_cn=TRUE
_search_str=""

# maskierung der Klammern, sonst error im IBM script "runmqakm"

     _cert_label=$(echo $1|sed "s/(/\\\(/g;s/)/\\\)/g")

     if [ $mqversion -ge 710 ] ; then
		     _search_str="Subject :"
	 else
     		 _search_str="Subject      : "
	 fi
     _cert_info=$($SSLCOMMAND -cert -details -db $KEYDB -stashed -label $_cert_label | grep -e "${_search_str}")
     IFS="," read a[1] a[2] a[3] a[4] a[5] a[6] a[7] a[8] <<< "${_cert_info#${_search_str}}"
     i=0
     while [[ $i -le 8 ]]
     do
        ((i=i+1))
        b=$(echo ${a[$i]}| cut -d'=' -f1)
        if [[ "$b" = "CN" ]];then
           _cert_name="$(echo ${a[$i]}| cut -d'=' -f2)"
           _no_cn=FALSE
           if [[ "$(echo $_cert_name|wc -w)" = "1" ]];then
             printf "%s" "$_cert_name"
           else
             printf "\"%s\"" "$_cert_name"
           fi
           i=9
        fi
     done
      if [[ "$_no_cn" = "TRUE" ]];then
       printf "%s" "n/a"
      fi
    return
}


#get label from certname
get_label_name(){

_cert_name=""
_cert_label=""

	$SSLCOMMAND -cert -list -db $KEYDB -stashed | grep -e "\!" -e "-" |grep -v "#" | cut -d$'\t' -f2 | while read _cert_label
	do
                 _cert_name=$(get_cert_name "$_cert_label")
                 if [[ "$_cert_name" = "$1" ]];then
                    printf "%s" "$_cert_label"
                 fi
	done
}


#show / delete expired certificate's
exp_certs(){
      _opt=$1  
      _today=$(date +%Y%m%d)
      $SSLCOMMAND -cert -list -db $KEYDB -stashed | grep -v "Certificates found" | grep -v "*" >${ssl_exp_filename}
      while read label
      do
               label=$(echo $label | sed 's/-!/!/g')
               label=$(echo ${label#!})
               label=$(echo ${label#-})            
               label=$(echo $label | sed 's/"//g')
               l1=""
               l1=$($SSLCOMMAND -cert -details -db $KEYDB -stashed -label "$label" 2>/dev/nul|grep -e "Label")
               l3=$(echo ${label#Label : })
               l2=""
               l2=$($SSLCOMMAND -cert -details -db $KEYDB -stashed -label "$label" 2>/dev/nul|grep -e "To ")
# for Version >7011
               if [[ "$?" -gt 0 ]];then
                   l2=$($SSLCOMMAND -cert -details -db $KEYDB -stashed -label "$label" 2>/dev/nul|grep -e "Not After")
                   l4=$(echo ${l2#Not After : })
               else
                   l4=$(echo ${l2#To           : })
               fi
               l5=$(echo "$(echo $l4 | awk '{print $1FS$2FS$3FS$4}')")
               if [[ "$l5" = "" ]];then l5="99991231"; fi
               l4=`date -d "$l5" +%Y%m%d`
               case $_opt in
                delete)
                  if [[ "$l4" -lt "$_today" ]];then
                        $SSLCOMMAND -cert -delete -db $KEYDB -stashed -label "$label"
                  fi
                  ;;
                show)
                	if [[ ! "$cmd_flg" == "TRUE" ]]
    				then 
                    	printf "%-8s %-50s %-50s %-s\n" ${qmgr} "\"$l3\"" "\"$(get_cert_name "$l3")\"" "$l4"
                	else
# add seperation for csv import in excel
                    	printf "%s;%s;%s;%s\n" ${qmgr} "$l3" "$(get_cert_name "$l3")" "$l4" 
                    fi                    
                     ;;
               esac
      done < ${ssl_exp_filename}
      
      if [[ -f ${ssl_exp_filename} ]];then
         rm ${ssl_exp_filename}
      fi
} 

make_backup() {
    _timestamp=`date +"%Y%m%d%H%M%S"`
    filename=/var/mw/mqbackup/ssl_${_timestamp}_${qmgr}.tar.gz
    echo "Start Backup to $filename"
	tar -cvzf ${filename} ${QMDATA}/${SSLDIR}
}

 # Wurde der Queue-Managername uebergeben ?
 if [ $# -lt 1 ]
 then
         ${su_cmd_pre} "${MQ_HOME_LATEST}/bin/dspmq"
         echo
         echo "Queue Manager :\c"
#         read QMGRINP
         read -t 600 QMGRINP || exit 1
         typeset -u qmgr=${QMGRINP}
         typeset -l qmgr_lowercase=${QMGRINP}
 else
         typeset -u qmgr=$1
         typeset -l qmgr_lowercase=$1
         if [ $# -gt 1 ]
         then
                 case $2 in
                 16)     cmd_flg=TRUE
                		 cmd_par=$2
                               ;;
                 14)     cmd_flg=TRUE
                         cmd_par=$2
                         cmd_par1=$3
                               ;;
                 13)     cmd_flg=TRUE
                           cmd_par=$2
                           cmd_par1=$3
                               ;;
                 6)      cmd_flg=TRUE
                           cmd_par=$2
                           cmd_par1=$3
                               ;;
                 3)      cmd_flg=TRUE
                           cmd_par=$2
                           cmd_par1=$3
                           cmd_par2=$4
                               ;;
                 g)      cmd_flg=TRUE
                           cmd_par=$2
                           cmd_par1=$3
                           cmd_par2=$4
                           cmd_par3=$5
                           case $6 in
                               norefssl) cmd_par4=no
                                     ;;
                               *)   cmd_par4=yes
                                    ;; 
                               esac 
                               ;;
                   f|r|d)  cmd_flg=TRUE
                           cmd_par=$2
                               ;;
                   *)       syntax
                           exit 1
                   ;;
             esac
         fi
 fi

# Checks and set multiversion vars
mqsetenv -m $qmgr

 # Select SSL MQ Command depending of MQ Version

 if [[ ! -z $mqversion ]] ; then
    if [ $mqversion -ge 710 ] ; then
       SSLCOMMAND="${MQ_HOME}/bin/runmqakm"
    elif  ${su_cmd_pre} "[ -x \"$( which gsk7capicmd 2>/dev/null )\" ]" ; then
       SSLCOMMAND="$( which gsk7capicmd )"
    fi
 fi

 if [ -d "/MQHA" ] ; then
         # fuer MQ Vers 7 data Verzeichnis zuweisen
     unset QMDATA
     . /usr/local/mw/mqm/tools/setlocals_${qmgr}
     if [[ -z "$QMDATA" ]]; then
            QMDATA=/MQHA/${qmgr}/data/qmgrs/${qmgr}
         fi
 else
     QMDATA=/var/mqm/qmgrs/${qmgr}
 fi

 SSLDIR=ssl
 KEYDB=$QMDATA/$SSLDIR/key.kdb


 while true
 do 
  if [[ "$cmd_flg" != "TRUE" ]]
  then 
   echo $cls$off
   p_logo "MQ SSLSetup               `date '+%d.%m.20%y %H:%M'` "
   p_line
   echo
   p_title "                  Version $SCRIPTVERSION   "
   echo " "
 #  p_line 
   
   echo "             Queuemanager: ${bold}${qmgr}${off}          "
   echo "                   Key DB: ${bold}[$KEYDB]${off}       "
   p_line
   p_fehler "\nHinweis:   $fehler"
   
   echo " "  
   echo " "  
   echo "$bold    1 $off      Show GSKit version               "
   echo "$bold    2 $off      Create KeyDB                     "      
   echo "$bold    3 $off      Import Root CA                   "
   echo "$bold    4 $off      Import all Daimler Root CAs      "
   echo "$bold    5 $off      Create Certificate Request       "      
   echo "$bold    6 $off      Import signed Certificate        "
   echo "$bold    7 $off      Show Certificates                "
   echo "$bold    8 $off      Show all Certificate Requests    "
   echo "$bold    9 $off      Show detailed Certificate Request" 
   echo "$bold   10 $off      Renew Certificate Request        " 
   echo "$bold   11 $off      Show Certificate Details         " 
   echo "$bold   12 $off      Delete Certificate Request       "
   echo "$bold   13 $off      Remove Certificate               "
   echo "$bold   14 $off      Change KeyDB Password incl. Expiry"
   echo "$bold   15 $off      Duplicate KeyDB                  "
   echo "$bold   16 $off      Show Password expire date        "
   echo "$bold    c $off      Change KeyDB                     "
   echo "$bold    a $off      Activate Duplicated KeyDB        "
   echo "$bold    r $off      Refresh Security Type (SSL)      "
   echo "$bold    d $off      Show Certificate expire date     "
   echo "$bold    f $off      Remove all expired Certificate   "
   echo "$bold    g $off      Replace Certificate              "
   echo "$bold    ! $off      Unix-Shell                       "              
   echo "$bold    x $off      Exit                             "    
   echo " "  
   p_line
   prompt
   
 # Auswahl einlesen  
#   read auswahl
   read -t 600 auswahl || exit 1
  else
      auswahl=${cmd_par}
  fi
  
   fehler=" " 
   case $auswahl in
   
     
   1) # show version
	if [[ ! -z $mqversion ]] ; then
    	if [ $mqversion -ge 710 ] ; then
    		$SSLCOMMAND -version
    	elif  ${su_cmd_pre} "[ -x \"$( which gsk7ver 2>/dev/null )\" ]" ; then
    		gsk7ver
    	fi
	fi   
    hit
    ;;
    
   2) # create keydb
        
      ${su_cmd_pre} "$SSLCOMMAND -keydb -create -db $KEYDB -pw password -type cms -expire 1095 -stashpw"
      chown mqm:mqm $QMDATA/$SSLDIR/*
      hit
      ;;
           
   3)  
  
      if [[ "$cmd_flg" == "TRUE" ]]
      then 
        LAROOTCA=$cmd_par1
      	FNROOTCA=$cmd_par2
      else
      	ls -1 $QMDATA/$SSLDIR/
        for indir in $indir_1 $indir_2 ;do
         if [[ -d $indir ]] ;then
            ls -1 $indir|while read f;do echo "$indir/$f";done
         fi
        done
        echo " "
      	echo "Please enter the file name of Root CA :\c"   
#      	read FNROOTCA
        read -t 600 FNROOTCA || exit 1
	    echo "Please enter the label of Root CA :\c"   
#      	read LAROOTCA
        read -t 600 LAROOTCA || exit 1
      fi   
      if [ -z $FNROOTCA ] || [ -z $LAROOTCA ]
      then
          echo 
          echo "no file name or label entered!!!"
          echo
      else
          if [[ ! $FNROOTCA = /* ]]; then
              FNROOTCA=${QMDATA}/"/${SSLDIR}/"$FNROOTCA
          fi
          $SSLCOMMAND -cert -add -db $KEYDB -stashed -label $LAROOTCA -file $FNROOTCA
      fi
  
      hit
      ;;

  4) 
# Import root certificates
         n=${#ROOT_CA[*]}
         i=0
         while ((i<$n))
         do
            FNROOTCA=${ROOT_CA[$i]}
            # remove dir and extention .cer
            LAROOTCA=`basename "${FNROOTCA}"`
            LAROOTCA=${LAROOTCA%%.cer}
            echo "Importing: \"$FNROOTCA\" with label \"$LAROOTCA\""
            $SSLCOMMAND -cert -add -db $KEYDB -stashed -label "$LAROOTCA" -file "$FNROOTCA" -format binary
            RC=$?
            if [ $RC = "0" ]; then
               $SSLCOMMAND -cert -details -db $KEYDB -stashed -label "$LAROOTCA"
            else
               if [ $RC = "23" ]; then
                  echo "Already there. Skip import."
                  $RC=0
               fi
            fi
            echo ""
            let i=i+1
         done
#      $SSLCOMMAND -cert -add -db $KEYDB -stashed -label "Corp-Root-CA" -file ${indir_2}/Corp-Root-CA.cer -format binary
#      $SSLCOMMAND -cert -add -db $KEYDB -stashed -label "CorpDir.net-Intermediate-CA01" -file ${indir_2}/CorpDir.net-Intermediate-CA01.cer -format binary  
#      $SSLCOMMAND -cert -add -db $KEYDB -stashed -label "CorpDir.net-EMEA-Issuing-CA01" -file ${indir_2}/CorpDir.net-EMEA-Issuing-CA01.cer -format binary

      hit
      ;;


   5) 
      
      # select location
      n=${#CERT_SITE[*]}
      i=0
      while ((i<$n))
      do
         echo "     "$i"   Site ["${CERT_SITE[$i]}"]"
         let i=i+1
      done

      echo "Please select your site [0,1,...]: \c"
#      read SITE
        read -t 600 SITE || exit 1
      
      echo "Please enter environment [PROD,INT,DEV]: \c"
#          read LENV
           read -t 600 LENV || exit 1

          case "$LENV" in
             "PROD"|"INT"|"DEV")
                CERTREQ=${QMDATA}/${SSLDIR}/request_ssl_${qmgr_lowercase}.arm
                CERT_CN=DAI.${CERT_SITE[$SITE]}.MQ.$LENV.${qmgr}
                CERT_DN=CN=$CERT_CN,OU=${CERT_OU[$SITE]},O=${CERT_O[$SITE]},ST=${CERT_ST[$SITE]},L=${CERT_L[$SITE]},C=${CERT_C[$SITE]}
                CERT_LABEL=ibmwebspheremq${qmgr_lowercase}

                echo "   Common name      : ["$CERT_CN"]"
                echo "   Site             : ["${CERT_SITE[$SITE]}"]"
                echo "   Organization unit: ["${CERT_OU[$SITE]}"]"
                echo "   Organization     : ["${CERT_O[$SITE]}"]"
                echo "   Location         : ["${CERT_L[$SITE]}"]"
                echo "   State            : ["${CERT_ST[$SITE]}"]"
                echo "   Country          : ["${CERT_C[$SITE]}"]"

                echo "Parameter are correct? [y|n]: \c"
#                read YN
                read -t 600 YN || exit 1

                case "$YN" in
                   "y"|"Y"|"yes"|"YES")
                       $SSLCOMMAND -certreq -create -db $KEYDB -stashed -label "$CERT_LABEL" -dn "$CERT_DN" -size 2048 -file "$CERTREQ"
                      RC=$?
                      if [ $RC != "0" ]; then
                         fehler="Error: Unexpected reason [$RC] on selection [$Select]!"
                      fi
                      chown mqm:mqm "$CERTREQ"
                      chmod 640 "$CERTREQ"
                      ;;
                   *)
                      fehler="Error: Certificate request aborted!"
                      ;;
                esac
                ;;
             *)
                fehler="Error: Unexpected environment!"
                ;;
          esac 

      hit
      ;;

      
   6) 
      if [[ "$cmd_flg" == "TRUE" ]]
      then 
      	FNRECVCA=$cmd_par1
      else
      	ls -1 $QMDATA/$SSLDIR/
      	echo "Please enter the certificate file name :\c"  
#      	read FNRECVCA
        read -t 600 FNRECVCA || exit 1
      fi
      if [ -z $FNRECVCA ] 
      then
         echo 
         echo "no file name entered!!!"
         echo
      else
      	$SSLCOMMAND -cert -receive -file $QMDATA/$SSLDIR/$FNRECVCA -stashed -db $KEYDB
      	chown mqm:mqm $QMDATA/$SSLDIR/*
      fi
	  if [[ ! "$cmd_flg" == "TRUE" ]]
      then
	  	   echo "Refesh Security? [y|n]: \c"
#           read YN
            read -t 600 YN || exit 1
      fi
                  case "$YN" in
                   "y"|"Y"|"yes"|"YES"|"Yes")
                      refresh_security
                      ;;
                   "n"|"N"|"no"|"NO"|"No")
                      echo "No Refesh Security"
                      ;;
                   *)
                      echo "No Refesh Security"
                      ;;
                  esac
       hit
      ;;


   7) 
      $SSLCOMMAND -cert -list all -db $KEYDB -stashed
      echo  

      hit
      ;;


   8) 
      $SSLCOMMAND -certreq -list -db $KEYDB -stashed
      echo

      hit
      ;;


   9) 
      $SSLCOMMAND -certreq -list -db $KEYDB -stashed
      echo
      echo "Please enter Label of certificate :\c"  
#      read LABCERTRQ
      read -t 600 LABCERTRQ || exit 1

      $SSLCOMMAND -certreq -details -db $KEYDB -stashed -label $LABCERTRQ
      
      hit
      ;;

   10) 
      $SSLCOMMAND -certreq -recreate -db $KEYDB -stashed -label ibmwebspheremq${qmgr_lowercase} -target $QMDATA/ssl/request_ssl_${qmgr_lowercase}.arm         

      hit
      ;;
   
  11) 
      $SSLCOMMAND -cert -list -db $KEYDB -stashed
      echo
      echo "Please enter Label of certificate :\c"  
#      read LABCERT
      read -t 600 LABCERT || exit 1

      $SSLCOMMAND -cert -details -db $KEYDB -stashed -label $LABCERT
      
      hit
      ;;

  12) 
      $SSLCOMMAND -certreq -list -db $KEYDB -stashed
      echo
      echo "Please enter Label of certificate :\c"  
#      read LABCERTRQD
      read -t 600 LABCERTRQD || exit 1

      $SSLCOMMAND -certreq -delete -db $KEYDB -stashed -label $LABCERTRQD
      
      hit
      ;;

  13) 
      if [[ "$cmd_flg" == "TRUE" ]]
      then 
      	LABCERTD=$cmd_par1
      else
	      $SSLCOMMAND -cert -list -db $KEYDB -stashed
	      echo
	      echo "Please enter Label of certificate :\c"  
#	      read LABCERTD
          read -t 600 LABCERTD || exit 1
      fi
      if [ -z $LABCERTD ] 
      then
         echo 
         echo "no label entered!!!"
         echo
      else
      	$SSLCOMMAND -cert -delete -db $KEYDB -stashed -label "${LABCERTD}"
      fi
      hit
      ;;

 14)
	if [[ "$cmd_flg" == "TRUE" ]]
   	then
       	NEWEXPIRE=$cmd_par1
	else
		echo "Please enter the new expiry value (days) [1095]:\c"
#        read NEWEXPIRE
        read -t 600 NEWEXPIRE || exit 1
        echo "Please enter the new KeyDB password(invisible) :\c"
	fi
# chek if script called local or remote
        stty >/dev/null 2>&1
        stty_erg=$?
# invisible input on
        [[ $stty_erg -eq 0 ]] && stty -echo
#        read NEWKEYDBPWD
        read -t 600 NEWKEYDBPWD || exit 1
		if [[ ! "$cmd_flg" == "TRUE" ]]
   		then
			echo "Please re-enter the new KeyDB password(invisible) :\c"    
#    		read NEWKEYDBPWD1
             read -t 600 NEWKEYDBPWD1 || exit 1
			if [[ ! "${NEWKEYDBPWD1}" = "${NEWKEYDBPWD}" ]];then
				echo "ERROR    : KeyDB passwords not equal"				
# invisible input off
        		[[ $stty_erg -eq 0 ]] &&  stty echo
				break
			fi
		fi
# invisible input off
        [[ $stty_erg -eq 0 ]] &&  stty echo
        NEWEXPIRE=${NEWEXPIRE:-1095}
    	if  [[ ! "$cmd_flg" == "TRUE" ]];then
        	echo ""
		fi
    	$SSLCOMMAND -keydb -changepw -db $KEYDB -new_pw $NEWKEYDBPWD -expire $NEWEXPIRE -stashed 2>&1 1>/dev/null
    	rc=$?
	    	if [[ "${rc}" -eq "0" ]];then
	    		echo "$(date +%Y-%m-%d) on ${qmgr} Password of KeyDB ${KEYDB} changed : OK"
			else
				gsk_error=$($SSLCOMMAND -keydb -changepw -db $KEYDB -new_pw $NEWKEYDBPWD -expire $NEWEXPIRE -stashed | grep CTGSK | awk '{print $1}')
				echo "$(date +%Y-%m-%d) on ${qmgr} Password of KeyDB ${KEYDB} not changed : ERROR ${rc} $gsk_error" 
			fi
    	hit
    	;;


   15) # Duplicate KeyDB
   
         echo "Please enter the new end of keydb directory name (${QMDATA}/ssl_*): \c"
#         read NEWNAME
         read -t 600 NEWNAME || exit 1
         NEWSSLDIR=$QMDATA/ssl_$NEWNAME
         if [[ -d "$NEWSSLDIR" ]]; then
            echo "$NEWSSLDIR exists already"
         else
            echo "create $NEWSSLDIR "
            mkdir $NEWSSLDIR
            chown mqm:mqm $NEWSSLDIR
           
            cp -p $QMDATA/$SSLDIR/key.* $NEWSSLDIR
         fi
         
         hit
         ;;
  
    16)
# chek if script called local or remote
     # if [[ "$cmd_flg" == "TRUE" ]]
     # then
     #   stty >/dev/null 2>&1
     #   stty_erg=$?
# invisible input on
		#[[ $stty_erg -eq 0 ]] &&  stty -echo
       # read KEYDBPWD
# invisible input off
	#	[[ $stty_erg -eq 0 ]] &&  stty echo
       # $SSLCOMMAND -keydb -expiry -db $KEYDB -pw $KEYDBPWD 2>&1 1>/dev/null
       # rc=$?
       # if [[ "$rc" -eq "0" ]]
       # then
	#			erg_str=$($SSLCOMMAND -keydb -expiry -db $KEYDB -pw $KEYDBPWD | grep -i "password" | cut -d':' -f2)
#				if [[ ${erg_str} == " 0" ]]
#				then
#					echo -n "$(date +%Y-%m-%d);${qmgr};${KEYDB};OK;0; \n"
#				else	
#					echo -n "$(date +%Y-%m-%d);${qmgr};${KEYDB};OK;0;$(date --date "${erg_str}" +%Y-%m-%d)\n"
#				fi
#    	else
#				gsk_error=$($SSLCOMMAND -keydb -expiry -db $KEYDB -pw $KEYDBPWD | grep CTGSK | awk '{print $1}')
#    			echo -n "$(date +%Y-%m-%d);${qmgr};${KEYDB};ERROR;${gsk_error};-\n" 
#    	fi
#      else
#        $SSLCOMMAND -keydb -expiry -db $KEYDB
#      fi
#      hit
 #     ;;


#if [[ ! -f "${QMDATA}/${SSLDIR}/key.sth" ]]; then
if [[ ! -f "${QMDATA}/${SSLDIR}/key.kdb" ]]; then
#                echo "${QMDATA}/${SSLDIR}/key.sth couldn't be found" >> /opt/mw/mqm/test/a.csv
#                echo "${QMDATA}/${SSLDIR}/key.sth couldn't be found"
                 echo "${QMDATA}/${SSLDIR}/key.kdb couldn't be found"
                exit 1;
        fi
#        KEYDBPWD=$(/opt/mw/mqm/bin/unstash.pl ${QMDATA}/${SSLDIR}/key.sth)

#        $SSLCOMMAND -keydb -expiry -db $KEYDB -pw $KEYDBPWD 2>&1 1>/dev/null
         $SSLCOMMAND -keydb -expiry -db $KEYDB -stashed 2>&1 1>/dev/null
        rc=$?
        if [[ "$rc" -eq "0" ]]
        then
#                               erg_str=$($SSLCOMMAND -keydb -expiry -db $KEYDB -pw $KEYDBPWD | grep -i "password" | cut -d':' -f2)
                                erg_str=$($SSLCOMMAND -keydb -expiry -db $KEYDB -stashed | grep -i "password" | cut -d':' -f2)
				if [[ ${erg_str} == " 0" ]]
				then
                                        echo -n "$(date +%Y-%m-%d),${qmgr},${KEYDB},OK,0 \n" 

    	else
                                         echo -n "$(date +%Y-%m-%d),${qmgr},${KEYDB},OK,0,$(date --date "${erg_str}" +%Y-%m-%d)\n" 
                                      
                                       
    	fi
      else
                                gsk_error=$($SSLCOMMAND -keydb -expiry -db $KEYDB -stashed | grep CTGSK )
			#	echo $gsk_error
                                 echo -n "$(date +%Y-%m-%d),${qmgr},${KEYDB},ERROR,\"${gsk_error}\"\n" 
      fi
      # else
      #  $SSLCOMMAND -keydb -expiry -db $KEYDB
      # fi
      hit
      ;;
                                                        
                                                       
      a) # Activate Duplicated KeyDB
    
         (
              cd ${QMDATA}
              ls -d1 ssl*
         )
    
                 echo "Please enter the new active keydb directory [ssl]: \c"
#                 read newSSLDIR
                 read -t 600 newSSLDIR || exit 1

                 newSSLDIR=${newSSLDIR:-ssl}
                 SSLDIR=ssl
                 SSLarchDIR=${QMDATA}/ssl_old

         if [[ ! -d "${QMDATA}/${newSSLDIR}" ]]; then
            echo "$newSSLDIR don't exists"
         else
          
          if [[ -d "${QMDATA}/${SSLarchDIR}" ]]; then
            echo "Archiv $SSLarchDIR exists!"
            echo "Overwrite? [y|n]: \c"
#            read YN
            read -t 600 YN || exit 1

            case "$YN" in
                   "y"|"Y"|"yes"|"YES")
                     
                    rm -R ${SSLarchDIR}
                    mv ${QMDATA}/${SSLDIR} ${QMDATA}/${SSLarchDIR}
                    mv ${QMDATA}/${newSSLDIR} ${QMDATA}/${SSLDIR}
                      ;;
                   *)
                      ;;
                esac
          else
            mv ${QMDATA}/${SSLDIR} ${SSLarchDIR}
            mv ${QMDATA}/${newSSLDIR} ${QMDATA}/${SSLDIR}
          fi
            
         fi

         refresh_security
                 
         KEYDB=${QMDATA}/${SSLDIR}/key.kdb
                 
         hit
         ;;

      c) # Change KeyDB
    
         (
            cd $QMDATA
            ls -d1 ssl*
         )
    
                 echo "Please enter the new keydb directory [ssl]: \c"
#                 read SSLDIR
                 read -t 600 SSLDIR || exit 1

                 SSLDIR=${SSLDIR:-ssl}
                 
                 KEYDB=$QMDATA/$SSLDIR/key.kdb
                 
                 hit
                 ;;
         
   !) p_title "Unix-Shell: With <exit> back to menue" 
      sh
      ;;     
  
   d)
     printf "$bold%-8s %-50s %-50s %-s$off\n" "qmgrname" "labelname" "CN=" "expiredate"
     exp_certs show
      hit
      ;;
      
   e) break 
      ;;

   f) 
      make_backup
      exp_certs delete
      hit
      ;;

   g) 
      if [[ "$cmd_flg" == "TRUE" ]]
      then 
      	    CERT2REP=$cmd_par1
      	    FNCA=$cmd_par2
      	    LACA=$cmd_par3
      	    YN=$cmd_par4
	  else
		    printf "$bold%-8s %-50s %-50s %-s$off\n" "qmgrname" "labelname" "CN=" "expiredate"
            exp_certs show
            echo "Please enter Name(CN) of certificate :\c"  
#		    read CERT2REP
            read -t 600 CERT2REP || exit 1
		    ls -1 $QMDATA/$SSLDIR
	        for indir in $indir_1 $indir_2 ;do
	         if [[ -d $indir ]] ;then
	            ls -1 $indir|while read f
	            do 
	            	echo "$indir/$f"
	            done
	         fi
	        done
	        echo " "
		    echo "Please enter the location(path and name) of the Certificate :\c"   
#		    read FNCA
            read -t 600 FNCA || exit 1 
		    echo "Please enter the label of the new Certificate :\c"   
#	      	read LACA
            read -t 600 LACA || exit 1  
      fi
      if [ -z $FNCA ] || [ -z $LACA ] || [ -z $CERT2REP ]; then
          echo 
          echo "no file name, certificat name or label entered!!!"
          echo
      else
          if [[ ! -f $FNCA ]]; then
	          echo 
	          echo "file  '$FNCA'  not found!!!"
	          echo
          else
    		  if [[ "$(echo $CERT2REP|wc -w)" -gt "1" ]];then 
    		  		CERT2REP="\"$CERT2REP\""
    		  fi
    		  if [[ "$(echo $LACA|wc -w)" -gt "1" ]];then 
    		  		LACA="\"$LACA\""
    		  fi
     		    make_backup
    		    LAB2REP=$(get_label_name "${CERT2REP}")
                if [[ ! -z $LAB2REP ]]; then
	              $SSLCOMMAND -cert -delete -db $KEYDB -stashed -label $LAB2REP
	              $SSLCOMMAND -cert -add -db $KEYDB -stashed -label $LACA -file $FNCA -format binary
	              if [[ ! "$cmd_flg" == "TRUE" ]]
                  then
	              	   echo "Refesh Security? [y|n]: \c"
#                       read YN
                        read -t 600 YN || exit 1
                  fi
                  
                  case "$YN" in
                   "y"|"Y"|"yes"|"YES"|"Yes")
                      refresh_security
                      ;;
                   "n"|"N"|"no"|"NO"|"No")
                      echo "No Refesh Security"
                      ;;
                   *)
                      echo "No Refesh Security"
                      ;;
                  esac
                else
                	echo "no label found for ${CERT2REP}"
                fi
          fi
      fi	      
      YN=""
      hit
      ;;

   r) # REFRESH SECURITY
      refresh_security
      hit
      ;;

   x) exit 
      ;;

      
   ?) fehler="Invalid input!"
      ;;

   *) fehler="Invalid selection!" 
      ;;

   esac
 done
