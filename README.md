#! /bin/ksh
#ifixdir=/opt/mw/mqm/ifix/800
#MQ_VERSION_LATEST=800

if [ $# -ne 2 ]; then
    echo "Usage : interimFixInstall.sh <MQ version e.g - 800 or 900> <Patchlevel e.g - 8007 or 9002>"
    echo "Example : interimFixInstall.sh 800 8007 or interimFixInstall.sh 900 9002"
    exit 1
fi

version=$1
patchlevel=$2
ifixpath=/opt/mw/mqm/ifix
ifixextrpath=/tmp/extract
mqinst=/opt/mqm${version}
[[ -d $ifixextrpath ]] || mkdir  $ifixextrpath
#if [[ $version -eq "800" ]]
#then
        fixpath=$ifixpath/$version
        #new
        mqxravail=$(rpm -qa | grep -i MQSeriesXRService | wc -l)
        if [[ $mqxravail -ne 0 ]]
        then
            fixpath="$ifixpath/$version/mqxr"
        fi
        numberoffiles=`ls -1 $fixpath/*.gz 2> /dev/null | wc -l`
        if [[ $numberoffiles -eq 0 ]]
        then
                echo "No Ifix to install"
                exit 0;
        fi
        echo "+++++++++++++++++++++++++++++++++++++++++++++++"
        echo "BEGIN THE EXTRACTION OF IFIX ZIP FILES"
        echo "+++++++++++++++++++++++++++++++++++++++++++++++"
        ctr=0

        for file in $(ls -1 $fixpath/*.gz 2> /dev/null)
        do
                fixfile=$(basename $file)
                fixfilelevel=$(echo ${fixfile} | awk -F "-" '{print $1}' | sed 's/\.//g')
                if [[ ${fixfilelevel} -eq ${patchlevel} ]]; then
                tar -zxf $file -C $ifixextrpath
                rc=`echo $?`
                if [[ $rc -eq 0 ]]
                then
                        echo "$rc Extraction of $file is successful"
                else
                        echo "$rc Extraction of $file is unsuccessful"
                fi
                else
                        echo "$file is not applicable for the patchlevel $patchlevel"
                fi
                ctr=`expr $ctr + 1`
                if [[ $ctr -ne $numberoffiles ]]
                then
                        echo "++++++++++++NEXT FILE EXTRACTION+++++++++++++++"
                else
                        echo "+++++++++++++NO MORE FILES TO EXTRACT++++++++++"
                fi
        done
ls -1 /tmp/extract/
        echo "+++++++++++++++++++++++++++++++++++++++++++++++"
        echo "BEGIN THE INSTALLATION OF INTERIM FIX"
        echo "+++++++++++++++++++++++++++++++++++++++++++++++"
        numberoffiles=`ls -1 $ifixextrpath 2> /dev/null | wc -l`
        ctr=0
        for file in $(ls -1 $ifixextrpath)
        do
                ifix=`echo $file | cut -c24-`
                echo $ifix
                grep -i $ifix $mqinst/mqpatch.dat
                rc=`echo $?`
                if [[ $rc -eq 0 ]]
                then
                        echo "Patch Already Installed"
                else
                cd $ifixextrpath/$file
                echo "+++++++++BEGIN TESTINSTALL FOR $file++++++++++"
                $ifixextrpath/$file/mqfixinst.sh $mqinst testinstall $file
                rc=`echo $?`
                echo $rc
#                ctr=`expr $ctr + 1`
                if [[ $rc -ne 0 ]]
                then
                        echo "++++++++TESTINSTALL FAILED for $file, DON'T PROCEED WITH FINAL INSTALLATION, check the reason in the output+++"
                        echo "+++++++++++++NEXT TESTINSTALL++++++++++++++++++"
                else
                        echo "+++++++TESTINSTALL SUCCESSFUL+++++++++++++++++"
                        echo "+++++++++++MQ COMMAND LEVEL BEFORE INSTALLING INTERIM FIX++++++++++++++++++++++"
                        $mqinst/bin/dspmqver | grep -i level
                        echo "+++++PROCEED WITH FINAL INSTALLATION++++++++++"
                        $ifixextrpath/$file/mqfixinst.sh $mqinst install $file
                        rc=`echo $?`
                        echo $rc
                        if [[ $rc -ne 0 ]]
                        then
                                echo "++++++++FINAL INSTALL FAILED FOR $file, check the reason in the output+++"
                                tail -10 $mqinst/mqpatch.log | grep -i FAILED
                                exit 1;
                        else
                                echo "++++++++FINAL INSTALLATION SUCCESSFUL FOR $file++++++++++++++++++++++"
                                echo "+++++++++++MQ COMMAND LEVEL AFTER INSTALLING INTERIM FIX++++++++++++++++++++++"
                                $mqinst/bin/dspmqver | grep -i level
                                echo "+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++"
                        fi
                fi
                ctr=`expr $ctr + 1`
                if [[ $ctr -eq $numberoffiles ]]
                then
                 echo "+++++++++++++NO MORE FILES TO INSTALL++++++++++"
                fi
                fi
        done
        echo "+++++++++++++++++++++++++++++++++++++++++++++++"
        echo "MQ Command Level Version is:"
        $mqinst/bin/dspmqver | grep -i level
#fi
rm -rf /tmp/extract/*
