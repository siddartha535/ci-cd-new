#!/bin/bash

PROG=`readlink -f $0`

if [[ $# -eq 0 || $# -ne 2 ]]
then
        echo "Usage: $PROG -all -json"
fi

if [[ $1 == "-all" ]]
then
        qmgrs=$(dspmq | grep -i "Running" | sed 's/)/(/g' | awk -F "(" '{print $2}')
        qmgrcount=$(dspmq | grep "Running" | wc -l)
fi

#function to get the number of threads per Queue Manager
get_no_of_threads() {
		#get the process id's for the Queue Manager
        value=$(ps -ef | grep $1 | grep -v grep | awk -F " " '{print $2}')
        threadcount=0
        for val in $value
        do
        		#get the number of threads for the PID
                #threads=$(ps -e -T | grep $val | grep -v grep | wc -l)
                threads=0
                threads=$(ps -e -T | awk '{ if ($1 == '$val') { print } }' | wc -l)
                #threadcount=`expr $threadcount + $threads`
                let threadcount=${threadcount}+${threads}
        done
        echo $threadcount
}

#function to get the shared memory segments per Queue Manager
get_shared_memory_bytes(){
        bytescount=0
        #get the process id's for the Queue Manager
        value2=`ps -ef | grep $1 | grep -v grep | awk '{print $2}'`

        for val2 in $value2
        do
				#get the semaphores id for the processes
                semid=$(ipcs -p | awk '{ if ($3 == '$val2') { print } }' | awk '{print $1}')
                for sem in $semid
                do
                		#get the amount of shared memory segments in bytes for each semaphore Id
                        by=$(ipcs -a |  awk '{ if ($2 == '$sem') { print } }' | awk '{print $5}')
                        #bytescount=`expr $bytescount + $by`
                        let bytescount=${bytescount}+${by}
                done
        done
        echo $bytescount
}

print_data(){
        echo "{\"qmgr\": \"$1\",\"shm\":" `get_shared_memory_bytes $1`",\"threads\":" `get_no_of_threads $1`"}"
}

print_json_format_data(){
		ctr=0
		echo "{\"qmgrs\": ["
		for qmgr in $qmgrs
		do
        		#ctr=`expr $ctr + 1`
        		let ctr=${ctr}+1
        		if [[ $ctr != $qmgrcount ]]
        		then
                		echo `print_data $qmgr`","
        		fi
        		if [[ $ctr == $qmgrcount ]]
        		then
                		print_data $qmgr
        		fi
		done
		echo "]}"
}

if [[ $2 == "-json" ]]
then
        print_json_format_data
fi
