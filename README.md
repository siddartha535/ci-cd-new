
    echo -e "${GREEN}Step:26 Stopping the qmgr/qmgrs ${NC}"
    for qmgr in ${qmgrs}
    do
        /opt/mw/mqm/bin/mqstop.sh -all
        sleep 10
      #  [[ "$(ps -ef | grep ${qmgr} |grep -v grep | wc -l)" -eq 0 ]]
        check_rc $? "Qmgr/qmgrs are in stopped state.Proceed in rebooting the server." "Exiting the program as the qmgrs are not in stopped state" "${counter}" "${qmgr}"
    done

}

echo "Time taken for MQv9.1 Migration is $SECONDS"
# End of migration function

echo -e "${GREEN}Check  MQ9.1 is installed ${NC}"
mqversion=$(dspmqver -i | grep '9.1.' | grep -i Version | awk -F':' '{printf $2}')
if [ ! -z $mqversion ]; then
    migrating_qmgrs
else
    RC=$(( RC + $? ))
    printError "Exiting the program as the MQv9.1 is not installed... Please execute the script mq91install.sh in bu-master to install MQv9.1 "
    echo RC=${RC}
    exit $RC
fi
