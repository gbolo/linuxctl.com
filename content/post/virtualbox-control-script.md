---
categories:
  - "administration"
  - "virtualization"
keywords:
  - "virtualbox"
tags:
  - "virtualbox"
  - "script"
  - "bash"
date: "2014-07-02"
title: "Virtualbox Control Script"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-virtualbox.png"

---

Having been a heavy [VirtualBox](https://www.virtualbox.org/) user while under Linux, I once came across this useful interactive script that I could run in my drop-down terminal that would easily allow me to manage my VMs. So useful was it, that it was aliased on all workstations i used, and carried around in my dropbox account. Here is a screenshot of it in action:
<!--more-->

![vbox script](/images/posts/virtualbox-script-output.png)


The code:


{{< codeblock "vboxctl.sh" "bash" "http://www.enide.net/webcms/index.php?page=vbctl" "vboxctl.sh" >}}
#!/usr/bin/env bash
##################################################################################################
# vbctl.sh
# VirtualBox Control
# This script scans and shows information on all virtual machines installed on the current host
# allows to perform basic tasks (start, pause, stop, etc)
# Designed particularly for headless machines
#
# Version 1.0 - May 2, 2011
# Version 1.1 - July 2011   - Enable VRDE activation, improve presentation.
# Version 1.2 - August 2011 - Fix machine listing when having sharedfolders.
# Version 1.3 - August 2011 - Using UUID instead of name for all functions.
#                             Correctly handle machine names with spaces
#                             Code cleaning and ANSI colors on menus.
#                             Animation while reading machines status.
#                             New option to change VRDE port.
#                             Removal of temporary files on exit.
#
# Version 1.4 - September 2011 - Added more color to main menu (guru meditation)
#                                Added option to mount and unmount cd/dvd images
#                                Grouped actions in commands menu
#
# Version 1.5 - September 2011 - Minor fix to list iso and img files for floppy
#                                controllers.
#
# Version 1.6 - October 2012   - Incorporating changes by Lance Norskog from Redwood City, CA
#                                 - Replacing 'seq' with a function to make it run under MacOS X
#                                 - Changed interpreter to /usr/bin/env to make the script more portable
#                                Minor cosmetic changes in VRDE and Load/Eject functions of the VM menu
#                                
# Version 1.7 - January 2013   - Minor fix in seq function to prevent printing an extra delimiter char
#
# Version 1.8 - November 2013  - Incorporating new features made by Sofia Perez
#                                 - Clone VM
#                                 - Destroy VM
#                                
# Version 1.9 - November 2013  - Colored progress bar when loading VM data on TTY that support color
#                                Support for dumb terminals without color (rotating wheel and vm state report)
#
#
# Developed on Oracle Virtual Box 4.0.4 on linux
# Tested on Linux and MacOS X
#
#
# List of contributors:
#
# Carlos Acosta Alamo
#    Original versions 1.0 .. 1.2
#    http://oracleexamples.wordpress.com
#    Oracle DBA
#
# Joao Figueiredo
#    Version 1.3 .. 1.5, 1.7, 1.9
#    http://www.enide.net
#
# Lance Norskog
#    Version 1.6
#    http://
#
# Sofia Perez
#    Version 1.8
#    http://
#
#
# Licensed under Creative Commons Attribution 3.0 Unported (CC BY 3.0)
# http://creativecommons.org/licenses/by/3.0/
# http://creativecommons.org/licenses/by/3.0/legalcode
#
###################################################################################################

## Current version
VERSION=1.9

## Virtualbox suggested VRDE ports
MIN_VRDE_PORT=5000
MAX_VRDE_PORT=5100

#
## Enable color only on these terminal types
#
## You can add more if you know they support color. Just 'echo $TERM' in your shell and add
## the terminal name to this list as another '-o "$TERM" = "terminalName"'
#
if [ "$TERM" = "ansi" -o "$TERM" = "xterm" -o "$TERM" = "linux" -o "$TERM" = "screen" ]; then
  ## ANSI foreground colors
  NORMAL="\033[0m"

  DARKRED="\033[31m"
  RED="\033[31;1m"

  DARKGREEN="\033[32m"
  GREEN="\033[32;1m"

  DARKYELLOW="\033[33m"
  YELLOW="\033[33;1m"

  DARKBLUE="\033[34m"
  BLUE="\033[34;1m"

  DARKPURPLE="\033[35m"
  PURPLE="\033[35;1m"

  DARKCYAN="\033[36m"
  CYAN="\033[36;1m"

  WHITE="\033[37;1m"
  UNDERLINEWHITE="\033[37;4m"
  WHITEONRED="\033[37;41;1m"

  BAR_BACK_COLOR="\033[41m"
  BAR_FORE_COLOR="\033[47m"

  USE_COLOR=1
else
  ## No colors
  NORMAL=""

  DARKRED=""
  RED=""

  DARKGREEN=""
  GREEN=""

  DARKYELLOW=""
  YELLOW=""

  DARKBLUE=""
  BLUE=""

  DARKPURPLE=""
  PURPLE=""

  DARKCYAN=""
  CYAN=""

  WHITE=""
  UNDERLINEWHITE=""
  WHITEONRED=""

  BAR_BACK_COLOR=""
  BAR_FORE_COLOR=""

  USE_COLOR=0
fi

## wheel animation
WHEEL_INDEX=1
WHEEL_CHARS[0]="-"
WHEEL_CHARS[1]="\\"
WHEEL_CHARS[2]="|"
WHEEL_CHARS[3]="/"
function animate_wheel()
{
    # remove old char
    echo -e "\b\b\c"

    # print char
    echo -e "${WHEEL_CHARS[$WHEEL_INDEX]} \c"

    # increment index
    WHEEL_INDEX=`expr $WHEEL_INDEX + 1`
    if [ $WHEEL_INDEX -ge 4 ]; then WHEEL_INDEX=0; fi
}

# macos has no 'seq', had to replace it.
# $1 number of items (1... up to $1)
# $2 separating character
# SEQ is the output variable
SEQ=
function seqN() {
    SEQ=
    C=$1
    GAP="$2"
    N=1
    while [ $C -gt 0 ]
    do
        C=`expr $C - 1`

        if [ $C -gt 0 ]; then
            SEQ="$SEQ$N$GAP"
        else
            SEQ="$SEQ$N"
        fi

        N=`expr $N + 1`
    done
}

#$1 nbr of spaces
function print_spaces()
{
    #SAVEIFS=$IFS
    IFS="-"

    if [ $1 -le 0 ]; then
        return;
    fi
    seqN $1 " "
    SPACES=$(echo $SEQ | sed 's/[0-9]//g')
    echo -e -n ${SPACES}

    #IFS=$SAVEIFS
    unset IFS
}

# $1 extra string for header
function print_header()
{
    FREE_SPACE=55 # 80 - 23 for "VirtualBox Console 1.x" - 2 for []
    REQUIRED_SPCS=`expr ${FREE_SPACE} - ${#1} + 1`

    clear
    echo -e "${WHITE}VirtualBox Console ${VERSION}${NORMAL} \c"
    print_spaces ${REQUIRED_SPCS}
    echo -e "[$1]"
    echo "--------------------------------------------------------------------------------"
    echo ""
}


# progress bar functions
BAR_SIZE=40 # default

# print bar background on current line
# $1 text message to place before the progress bar
# $2 number of steps
function initialize_bar() {

    local i=0
    BAR_SIZE=`expr 80 - ${#1} - 1`

    # bar max steps and remaining chars due to the integer arithmetic
    bar_max=$2
    bar_cur=0
    bar_chars_left=`expr $BAR_SIZE % $bar_max`


    # output message without changing the line and leave a white space
    echo -n -e "$1 "

    seqN $BAR_SIZE " "

    echo -n -e ${BAR_BACK_COLOR}
    for i in $SEQ; do
        echo -n " "
    done
    echo -n -e ${NORMAL}

    # move cursor back to start
    for i in $SEQ; do
        echo -n -e "\b"
    done
}

function increment_bar() {

    local i=0

    let bar_cur=$bar_cur+1;
    let nbr_chars=$BAR_SIZE/$bar_max;

    # add one of the remaining chars (due to the integer arithmetics) every % chars_left of current pos
    if [ $bar_chars_left -gt 0 ]; then
        let nbr_chars=$nbr_chars+1;
        bar_chars_left=`expr $bar_chars_left - 1`
    fi

    local ifs_safe=$IFS
    IFS=" "

    seqN $nbr_chars " "
    for i in $SEQ; do
        echo -n -e "${BAR_FORE_COLOR} "
    done
    echo -n -e "${NORMAL}"

    IFS=$ifs_safe
}


function refresh_vmdetails ()
{
    print_header "scanning for virtual machines"
    #echo -e "Please wait - \c"

    VMDETAILS=.vbox_vmdetails
    TEMPFILE=.vbox_temp
    TEMPFILE2=.vbox_temp2

    local nbr_vms=`VBoxManage list vms|wc -l`
    if [ $USE_COLOR -gt 0 ]; then initialize_bar "Found $nbr_vms VMs, fetching details" "$nbr_vms"
    else echo -e "Found $nbr_vms VMs, fetching details - \c"
    fi

    # JP: IFS is the field separator in BASH, default is space, so I'm redefining it to \n
    # JP: so it's possible to iterate VM names with spaces in the for loop
    IFS=$(echo -en "\n\b")

    VM_GETINFO="Name:|Guest OS:|Memory size:|Number of CPUs:|State:|VRDE:|Hardware UUID:"
    index=1
    rm -f $VMDETAILS >/dev/null 2>&1

    # JP: redefined the awk field separator to \" to get the VM names with spaces
    for i in `VBoxManage list vms|awk -F\" '{print $2}'` ; do
        if [ $USE_COLOR -gt 0 ]; then increment_bar
        else animate_wheel
        fi

        VBoxManage showvminfo "$i"|egrep "$VM_GETINFO">$TEMPFILE2
        head -7 $TEMPFILE2 >$TEMPFILE
        while read vmdetail; do
            echo -n "myvm$index) "$vmdetail >>$VMDETAILS
            echo "" >>$VMDETAILS
        done<$TEMPFILE
        index=$(($index+1));
    done

    # JP: restore IFS state otherwise all echos print a lot of spaces
    unset IFS
}

function list_vm_details ()
{
    print_header "virtual machine control options"
    grep $1 $VMDETAILS|sed -e "s/"$1"/   /"|grep -v VRDE
    grep $1 $VMDETAILS|sed -e "s/"$1"/   /"|grep VRDE|sed -e 's/\(.*\) (\([A-Za-z0-9\. ]*\), \([A-Za-z0-9\ ]*\),.*/\1 (\2, \3)/'
}

function select_vm ()
{
response_valid=1
while [ $response_valid -gt 0 ];do
    print_header "select id from the list or exit with CTRL-C"
    for VMID in `grep "Name:" $VMDETAILS|awk '{print $1}'`
    do
        VMID=$VMID" "
        VMNAME=`grep $VMID $VMDETAILS|sed -e 's/myvm//'|grep "Name:"`
        VRDE=`grep $VMID $VMDETAILS|sed -e "s/${VMID}//g"|grep "VRDE:"|sed -e 's/, Multi.*//' -e 's/(//'`
        VMSTATE=`grep $VMID $VMDETAILS|sed -e "s/${VMID}//g"|grep "State:"|sed 's/(since.*//'`"  "

        if [ $USE_COLOR -eq 0 ]; then
          SHORTVMSTATE="- [`echo $VMSTATE | awk -F\: '{print $2}'|awk '{gsub(/^[ \t]+|[ \t]+$/,""); print;}'`] "
        fi


        case $VMSTATE in
            *"powered off"*) echo -e $DARKRED $VMNAME ${NORMAL}${SHORTVMSTATE}- $VRDE ;;
            *"aborted"*) echo -e $RED $VMNAME ${NORMAL}${SHORTVMSTATE}- $VRDE ;;
            *"running"*) echo -e $DARKGREEN $VMNAME ${NORMAL}${SHORTVMSTATE}- $VRDE ;;
            *"paused"*) echo -e $YELLOW $VMNAME ${NORMAL}${SHORTVMSTATE}- $VRDE ;;
            *"guru meditation"*) echo -e $WHITEONRED $VMNAME - ${VRDE} ${NORMAL}${SHORTVMSTATE} ;;
            *) echo -e $NORMAL $VMNAME - $VMSTATE ;;
        esac
    done # for

    No_VMs=`VBoxManage list vms|wc -l`

    echo ""
    echo -n "Select a VM Id: ";read SELECTED_VM
    if echo $SELECTED_VM | grep "^[0-9]*$">/dev/null
        then
          if [ $SELECTED_VM -gt 0 -a $SELECTED_VM -le $No_VMs ];then
              response_valid=0
         fi
    fi

done #while

SELECTED_VM=myvm$SELECTED_VM")"
SELECTED_VM_NAME=`grep $SELECTED_VM $VMDETAILS|grep "Name:"|awk -F\: '{print $2}'|awk '{gsub(/^[ \t]+|[ \t]+$/,""); print;}'`
SELECTED_VM_UUID=`grep $SELECTED_VM $VMDETAILS|grep "Hardware UUID:"|awk -F\: '{print $2}'`
#SELECTED_VM_STATUS=`grep $SELECTED_VM $VMDETAILS|grep "State:"|awk '{print $3}'`

list_vm_details $SELECTED_VM

echo "--------------------------------------------------------------------------------"
}


function STARTUP_VM ()
{
if  grep $SELECTED_VM $VMDETAILS|grep "VRDE:"|grep "enabled" >/dev/null
then
    echo "Starting up"
else
    read -s -n 1 -p "Warning, VRDE is not enabled, enable it? (Y/[N])" SETVRDE
    if [ $SETVRDE == "Y" ] || [ $SETVRDE == "y" ]
    then
        response_valid=1
        while [ $response_valid = 1 ];do
            echo
            read -p "Enter port number (between $MIN_VRDE_PORT and $MAX_VRDE_PORT): " PORTNUMBER
            if [ $PORTNUMBER -ge $MIN_VRDE_PORT -a $PORTNUMBER -le $MAX_VRDE_PORT ] >/dev/null 2>&1;then
                VBoxManage modifyvm $SELECTED_VM_UUID --vrde on
                VBoxManage modifyvm $SELECTED_VM_UUID --vrdeport $PORTNUMBER
                response_valid=0
            else
                echo "Invalid port number, please choose between $MIN_VRDE_PORT and $MAX_VRDE_PORT"
            fi
        done
    fi
fi
VBoxManage startvm  $SELECTED_VM_UUID --type headless
}

function change_vrde_port ()
{
response_valid=1
while [ $response_valid = 1 ];do
    echo ""
    read -p "Enter port number (between $MIN_VRDE_PORT and $MAX_VRDE_PORT, 0 to return): " PORTNUMBER
    if [ $PORTNUMBER -eq 0 ]; then
      echo "Nothing changed"
      sleep 1
      return;
    elif [ $PORTNUMBER -ge $MIN_VRDE_PORT -a $PORTNUMBER -le $MAX_VRDE_PORT ] >/dev/null 2>&1;then
        VBoxManage modifyvm $SELECTED_VM_UUID --vrde on
        VBoxManage modifyvm $SELECTED_VM_UUID --vrdeport $PORTNUMBER
        response_valid=0
    else
        echo "Invalid port number, please try again"
    fi
done
}

control_vm()
{
response_valid=1
while [ $response_valid -gt 0 ];do
    echo "Available Options"
    echo ""
    echo -e "  1) ${GREEN}STARTUP${NORMAL} (Powered off or Saved)      5) ${YELLOW}Reset${NORMAL}"
    echo -e "  2) ${DARKYELLOW}Pause${NORMAL}                               6) ${YELLOW}Save state${NORMAL}"
    echo -e "  3) ${DARKGREEN}Resume${NORMAL} (from paused)                7) ${YELLOW}Power Off${NORMAL}"
    echo -e "  4) ${DARKRED}ACPI power button${NORMAL}"
    echo -e ""
    echo -e "  8) ${PURPLE}Change VRDE port${NORMAL}                    10) ${DARKGREEN}Clone${NORMAL}"
    echo -e "  9) ${BLUE}Load/Eject removable media${NORMAL}          11) ${DARKRED}Destroy${NORMAL}"
    echo -e "\n  0) Select another machine"
    echo -n -e "\nOption: " ;read VM_ACTION

    if echo $VM_ACTION | grep "^[0-9]*$">/dev/null
     then
       if [ $VM_ACTION -ge 0 -a $VM_ACTION -le 11 ];then
        response_valid=0
       fi
    fi
done  #while

case  $VM_ACTION in
        1 ) STARTUP_VM
            echo -n "Press enter to continue..."; read dummy
            ;;
        2 ) VBoxManage controlvm $SELECTED_VM_UUID pause
            echo -n "Press enter to continue..."; read dummy
        ;;
        3 ) VBoxManage controlvm $SELECTED_VM_UUID resume
            echo -n "Press enter to continue..."; read dummy
        ;;
        4 ) VBoxManage controlvm $SELECTED_VM_UUID acpipowerbutton
            echo -n "Press enter to continue..."; read dummy
        ;;
        5 ) VBoxManage controlvm $SELECTED_VM_UUID reset
            echo -n "Press enter to continue..."; read dummy
        ;;
        6 ) VBoxManage controlvm $SELECTED_VM_UUID savestate
            echo -n "Press enter to continue..."; read dummy
        ;;
        7 ) VBoxManage controlvm $SELECTED_VM_UUID poweroff
            echo -n "Press enter to continue..."; read dummy
        ;;
        8 ) change_vrde_port
        ;;
        9 ) clear
            list_vm_details $SELECTED_VM
            echo "-------------------------------------------------------------------------------"
            ask_user_for_controller

            if [ ${opt} -ne 0 ]; then
              echo ""
              ask_user_for_mount_umount_action "$CTRL_NAME"
              echo ""
              case $opt in
                1) echo ""
                   mount_drive "$CTRL_NAME" "$CTRL_PORT" "$CTRL_DEVICE" "$SELECTED_VM_UUID"
                   echo -n "Press enter to continue..."; read dummy
                ;;
                2) echo ""
                   umount_drive "$CTRL_NAME" "$CTRL_PORT" "$CTRL_DEVICE" "$SELECTED_VM_UUID"
                   echo -n "Press enter to continue..."; read dummy
                ;;
                *) echo "Nothing changed"
                   sleep 1
                ;;
              esac
            else
              echo "Nothing changed"
              sleep 1
            fi

        ;;
        10 )  SELECTED_VM_STATUS=`grep $SELECTED_VM $VMDETAILS|grep "State:"|awk '{print $3}'`
          if [ $SELECTED_VM_STATUS != "powered" ] ; then
            echo -e "You must turn off the virtual machine."        
          else
            read -p "Enter name of new virtual machine (empty to cancel): " VM_NEW
            if [ "$VM_NEW" == "" ]; then
              echo "Nothing changed."
            else
              echo "Start cloning..."
              vboxmanage clonevm $SELECTED_VM_UUID --mode all --options keepnatmacs --name $VM_NEW --register
            fi
          fi

          echo -n "Press enter to continue..."; read dummy
        ;;
        11 )  SELECTED_VM_STATUS=`grep $SELECTED_VM $VMDETAILS|grep "State:"|awk '{print $3}'`
          if [ $SELECTED_VM_STATUS != "powered" ] ; then
            echo -e "You must turn off the virtual machine."        
          else
            read -s -n 1 -p "Are you sure to destroy the virtual machine \"$SELECTED_VM_NAME\"? (Y/[N]) " VM_DESTROY
            if [ "$VM_DESTROY" == "Y" ] || [ "$VM_DESTROY" == "y" ]; then
              echo -e "\nRemoving virtual machine \"$SELECTED_VM_NAME\"..."
              vboxmanage unregistervm $SELECTED_VM_UUID --delete
            else
              echo -e "\nNothing changed."
            fi
          fi

          echo -n "Press enter to continue..."; read dummy
        ;;

        0 ) ;;

esac
}




###############################################################################
###############################################################################
# self contained load/eject floppy/cd/dvd image functions
###############################################################################

# $1 lower limit, $2 upper limit, $3 prompt
# $opt is the output
function read_delimited()
{
  response_valid=1;
  while [ $response_valid -gt 0 ]; do

    read -p "$3" opt;
    [ "$opt" == "" ] && opt=-1;
    if [ $opt -ge $1 ] && [ $opt -le $2 ]; then
      response_valid=0;
    else
      echo "Invalid number, type a number between $1 and $2"
      echo ""
    fi

  done # while
}

# inputs: $SELECTED_VM_UUID, $SELECTED_VM_NAME
# outputs: $CTRL_NAME, $CTRL_PORT, $CTRL_DEVICE
function ask_user_for_controller()
{
  TMP3=.cdchangetmp.txt

  # get all vm info
  VBoxManage showvminfo ${SELECTED_VM_UUID} > ${TMP3}

  # change InternalFieldSeparator to \n
  IFS=$'\n'

  # get interfaces names
  total_ifaces=1
  for IFACE in `cat ${TMP3} | grep "Storage Controller Name" | awk -F: '{print $2}'` ; do
    controller[$total_ifaces]=${IFACE}
    total_ifaces=`expr $total_ifaces + 1`
  done

  # restore to (white spaces and similar)
  unset IFS;

  # decrement one in the counter so it represents the last index of the array controller
  [ $total_ifaces -gt 1 ] && total_ifaces=`expr $total_ifaces - 1`;

  # trim interface names
  seqN $total_ifaces " "
  for i in $SEQ; do
      controller[$i]=`echo ${controller[$i]}`;
  done


  # now get all controller ports for the interface and show them
  echo "Available interfaces on ${SELECTED_VM_NAME}"
  total_ports=1
  seqN $total_ifaces " "
  for i in $SEQ; do

    echo "${controller[$i]}"
    this_controller_nbr_ports=0;

    IFS=$'\n'
    for PORT in `cat ${TMP3} | grep -v "Storage Controller Name" | grep ${controller[$i]} | sed -e 's/(UUID:.*)//g'` ; do
      ports[$total_ports]=${PORT};
      echo "   $total_ports) ${ports[$total_ports]}"
      total_ports=`expr $total_ports + 1`;
      this_controller_nbr_ports=`expr $this_controller_nbr_ports + 1`;
    done
    unset IFS;

    if [ $this_controller_nbr_ports -eq 0 ]; then
      echo "   -) no devices found";
    fi

    echo "";

  done

  echo "   0) Do nothing and go back";
  echo "";

  # restore to (white spaces and similar)
  unset IFS;

  # delete the temporary file
  rm -f ${TMP3}

  # decrement one in the counter so it represents the last index of the array ports
  [ $total_ports -gt 1 ] && total_ports=`expr $total_ports - 1`;

  # ask for controller
  read_delimited 0 $total_ports "Option: "
  #echo "You selected $opt, ${ports[$opt]}"

  # get controller name and trim it
  CTRL_NAME=`echo ${ports[$opt]} | sed -e 's/\([A-Za-z0-9 ]*\)(\([0-9]*\), \([0-9]*\)): \(.*\)/\1/'`;
  CTRL_NAME=`echo $CTRL_NAME`;

  # get port and device
  CTRL_PORT=`echo ${ports[$opt]} | sed -e 's/\([A-Za-z0-9 ]*\)(\([0-9]*\), \([0-9]*\)): \(.*\)/\2/'`;
  CTRL_DEVICE=`echo ${ports[$opt]} | sed -e 's/\([A-Za-z0-9 ]*\)(\([0-9]*\), \([0-9]*\)): \(.*\)/\3/'`;

  #echo "Interface [${CTRL_NAME}]"
  #echo "Port      [${CTRL_PORT}]"
  #echo "Device    [${CTRL_DEVICE}]"
}

# inputs: $1 controller name
# outputs: $opt
function ask_user_for_mount_umount_action()
{
  echo "Action on $1"
  echo "  1) Load Floppy/CD/DVD image"
  echo "  2) Eject image"
  echo "  0) Do nothing and go back"

  read_delimited 0 2 "Option: "
}

# inputs: $1 controller name, $2 port, $3 device, $4 VM_UUID
# outputs: none
function umount_drive()
{
  # Example:
  # VBoxManage storageattach 1a03448b-baeb-4b3c-a811-e3b26d4be8e4 --storagectl "IDE Controller" --port 1 --device 0 --type dvddrive --medium emptydrive

  echo "Ejecting from $1"
  VBoxManage storageattach $4 --storagectl "$1" --port $2 --device $3 --type dvddrive --medium emptydrive
}

# inputs: $1 controller name, $2 port, $3 device, $4 VM_UUID
# outputs: none
function mount_drive()
{
  # ask for file name
  while true; do
    echo "Available disk images"
    list_of_isos=`ls -1 *.iso *.img *.ima 2> /dev/null`
    IFS=$'\n'
    for file in $list_of_isos; do
      echo "  $file";
    done
    unset IFS;
    echo " "

    read -p "Filename (no name to cancel): " filename;

    if [ "$filename" == "" ] ; then
      echo ""
      echo "Nothing changed";
      return;
    fi

    if [ -f "$filename" ] ; then
      break;
    else
      echo "$filename not found, check name and try again";
    fi

  done

  # mount
  echo "Loading $filename on $1"
  VBoxManage storageattach $4 --storagectl "$1" --port $2 --device $3 --type dvddrive --medium "$filename"
}

###############################################################################
# end of load/eject cd/dvd images
###############################################################################
###############################################################################



function cleanup_before_exit()
{
    echo -e "\nSee you soon...";
    rm -f ${VMDETAILS} ${TEMPFILE} ${TEMPFILE2}
}

####
#### MAIN
####

#set -x

if [ "`which vboxmanage`" == "" ] || [ "`which VBoxManage`" == "" ]; then
  echo -e "Error: `basename $0` requires VirtualBox utilities to be installed and available in the \$PATH\n       Make sure vboxmanage is available and try again.\n"
  exit 1
fi

trap "cleanup_before_exit;exit" SIGINT SIGTERM
clear
while [ 1 = 1 ];do
    refresh_vmdetails
    select_vm
    #read -s -n 1 -p "Press a key to continue.."
    control_vm
done
{{< /codeblock >}}
