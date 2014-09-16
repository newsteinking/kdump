Kdump Anlaysis
===================================

   kdump는 시스템이 깨졌을경우에 시스템 코아 덤프를 받는 툴로 사용된다.
   이러한 캡쳐된 코아 덤프는 시스템 오류등을 분석하고 향후에 시스템 크래쉬를 재발하지 않도록
   수정하는데 사용되어 진다.
   kdump는 crashkernel이라는 작은 부차적인 메모리 영역을 할당한다.
   이 부수적 또는 크래쉬 커널은 시스템이 크래쉬되었을때마다 코어 덤프 이미지를 받는데 사용된다.





1. Install Kdump Tools
------------------------

   일단 kexec-tools 패키지의 일부인 kdump를 인스톨한다.

    ::

    $ yum install kexec-tools




2. Set crashkernel in grub.conf
--------------------------------

   일단 패키지가 인스톨된후 /boot/grub/grub.conf 파일을 수정한다.

    ::

    $crashkernel=auto  nmi_watchdog=1


   만약 메모리가 2G보다 작으면
    crashkernel=128M nmi_watchdog=1

   만약 메로리가 2G보다 크다면
    crashkernel=auto  nmi_watchdog=1




3. Configure Dump Location
--------------------------------

   커널이 크래쉬가 발생하게 되면 /etc/kdum.conf 화일에 설정된 내용에 때라 로컬 파일시스템내지 리모트 NFS에 캡쳐된다.
   이것은 kexec-tools 패키지가 인스톨되었으면 자동으로 생성된다.

    ::

    # vi /etc/kdump.conf::
    #raw /dev/sda5
    #ext4 /dev/sda3
    #ext4 LABEL=/boot
    #ext4 UUID=03138356-5e61-4ab3-b58e-27507ac41937
    #net my.server.com:/export/tmp
    #net user@my.server.com
    path /var/crash
    core_collector makedumpfile -c --message-level 1 -d 31
    #core_collector scp
    #core_collector cp --sparse=always
    #extra_bins /bin/cp
    #link_delay 60
    #kdump_post /var/crash/scripts/kdump-post.sh
    #extra_bins /usr/bin/lftp
    #disk_timeout 30
    #extra_modules gfs2
    #options modulename options
    #default shell
    #debug_mem_level 0
    #force_rebuild 1
    #sshkey /root/.ssh/kdump_id_rsa


   상위 파일에서:
      raw 디바이스에 덤프를 쓰고자 한다면 "raw /dev/sda5" 코멘트를 없애고 실행하면 된다.


4. Configure Core Collector
--------------------------------

   다음과정은 kdump 설정에서 코아 콜렉터를 설정하는 것이다. 이것은 캡처된 데이터를 압축하고 캡처된 코아파일에서
   불필요한 정보를 필터링하는데 중요하다.


    ::

core_collector makedumpfile -c --message-level 1 -d 31

    makedumpfile specified in the core_collector actually makes a small DUMPFILE by compressing the data.
    makedumpfile provides two DUMPFILE formats (the ELF format and the kdump-compressed format).
    By default, makedumpfile makes a DUMPFILE in the kdump-compressed format.
    The kdump-compressed format can be read only with the crash utility, and it can be smaller than the ELF format because of the compression support.
    The ELF format is readable with GDB and the crash utility.
    "-c" is to compresses dump data by each page
    "-d" is the number of pages that are unnecessary and can be ignored.




5. Restart kdump Services
--------------------------------

  kdump가 설정되면 kdump 서비스를 재시작 한다.


    ::

    #chkconfig kdump on
    # service kdump restart
    Stopping kdump:   [  OK  ]
    Starting kdump:   [  OK  ]

    # service kdump status
    Kdump is operational

   만약 kdump 서비스 실행에 문제가 있으면 kdump 모듈이나 crashkernel 변수가 적절히 설정되어 있지 않은것이다.
   /proc/cmdline 을 확인하여 crashkernel 변수값이 제대로 설정되었는지 확인한다.



6. Manually Trigger the Core Dump
------------------------------------

   다음 명령을 통해 인위적으로 코어덤프를 만들 수 있다.

    ::

    echo 1 > /proc/sys/kernel/sysrq
    echo c > /proc/sysrq-trigger



   서버는 자동으로 리붓되고 크래쉬 덤프파일을 생성할 것이다.

7. View the Core Files
------------------------------------

   서버가 재시작 되었으면, 코아파일을 /var/crash/ 하위에 생성된 것을 볼 수 있을것이다.


    ::

    # ls -lR /var/crash
    drwxr-xr-x. 2 root root 4096 Mar 26 11:06 127.0.0.1-2014-03-26-11:06:43

    /var/crash/127.0.0.1-2014-03-26-11:06:43:
    -rw-------. 1 root root 33595159 Mar 26 11:06 vmcore
    -rw-r--r--. 1 root root    79498 Mar 26 11:06 vmcore-dmesg.txt



8. Kdump analysis using crash
------------------------------------

   kdump에 의해서 생성된 코아파일은 crash 툴을 사용한다.
   이것은 또한 netdump,diskdump,xendump등에 의해서 생성된 코아파일도 분석할 수 있다.

   아래와 같이 크래쉬 명령을 통해 시작한다.

    ::

    crash /var/crash/127.0.0.1-2014-09-16-14:47:55/vmcore  /home/sean/rpmbuild/BUILD/kernel-2.6.32-431.23.3.el6/
    linux-2.6.32-431.23.3.el6.x86_64/vmlinux



9. View the Process when System Crashed
------------------------------------------

   ps 명령은 시스템이 크래쉬되었을때 실행되고 있던 프로세스를 표시한다.

    ::


    crash> ps
    PID    PPID  CPU       TASK        ST  %MEM     VSZ    RSS  COMM
      0      0   0  ffffffff81a8d020  RU   0.0       0      0  [swapper]
      1      0   0  ffff88013e7db500  IN   0.0   19356   1544  init
      2      0   0  ffff88013e7daaa0  IN   0.0       0      0  [kthreadd]
      3      2   0  ffff88013e7da040  IN   0.0       0      0  [migration/0]
      4      2   0  ffff88013e7e9540  IN   0.0       0      0  [ksoftirqd/0]
      7      2   0  ffff88013dc19500  IN   0.0       0      0  [events/0]


10. View Swap space when System Crashed
------------------------------------------

   swap 명령은 시스템이 크래쉬되었을때 스왑 공간 영역을  표시한다.

    ::

    crash> swap
    FILENAME           TYPE         SIZE      USED   PCT  PRIORITY
    /dm-1            PARTITION    2064376k       0k   0%     -1


11. View IPCS when System Crashed
------------------------------------------


    ipcs  명령은 시스템이 크래쉬되었을때 공유 메모리 공간을  표시한다.

    ::

    crash> ipcs
    SHMID_KERNEL     KEY      SHMID      UID   PERMS BYTES      NATTCH STATUS
    (none allocated)

    SEM_ARRAY        KEY      SEMID      UID   PERMS NSEMS
    ffff8801394c0990 00000000 0          0     600   1
    ffff880138f09bd0 00000000 65537      0     600   1

    MSG_QUEUE        KEY      MSQID      UID   PERMS USED-BYTES   MESSAGES
    (none allocated)


12. View IRQ when System Crashed
------------------------------------------

    irq  명령은 시스템이 크래쉬되었을때 irq 상태를  표시한다.

    ::

    crash> irq -s
           CPU0
      0:        149  IO-APIC-edge     timer
      1:        453  IO-APIC-edge     i8042
      7:          0  IO-APIC-edge     parport0
      8:          0  IO-APIC-edge     rtc0
      9:          0  IO-APIC-fasteoi  acpi
     12:        111  IO-APIC-edge     i8042
     14:        108  IO-APIC-edge     ata_piix
 .
 .

    vtop – This command translates a user or kernel virtual address to its physical address.
    foreach – This command displays data for multiple tasks in the system
    waitq – This command displays all the tasks queued on a wait queue.


13. View the Virtual Memory when System Crashed
-------------------------------------------------


    vm  명령은 시스템이 크래쉬되었을때 가상 메모리 사용량을   표시한다.

    ::

    crash> vm
    PID: 5210   TASK: ffff8801396f6aa0  CPU: 0   COMMAND: "bash"
       MM              		 PGD          RSS    TOTAL_VM
    ffff88013975d880  ffff88013a0c5000  1808k   108340k
      VMA           START       END     FLAGS FILE
    ffff88013a0c4ed0     400000     4d4000 8001875 /bin/bash
    ffff88013cd63210 3804800000 3804820000 8000875 /lib64/ld-2.12.so
    ffff880138cf8ed0 3804c00000 3804c02000 8000075 /lib64/libdl-2.12.so


14. View the Open Files when System Crashed
-------------------------------------------------


    files  명령은 시스템이 크래쉬되었을때 열린 파일을    표시한다.

    ::
    crash> files
    PID: 5210   TASK: ffff8801396f6aa0  CPU: 0   COMMAND: "bash"
    ROOT: /    CWD: /root
    FD       FILE            DENTRY           INODE       TYPE PATH
     0 ffff88013cf76d40 ffff88013a836480 ffff880139b70d48 CHR  /tty1
      1 ffff88013c4a5d80 ffff88013c90a440 ffff880135992308 REG  /proc/sysrq-trigger
    255 ffff88013cf76d40 ffff88013a836480 ffff880139b70d48 CHR  /tty1
    ..


15. View System Information when System Crashed
-------------------------------------------------


    sys  명령은 시스템이 크래쉬되었을때 시스템정보를     표시한다.

     ::
    crash> sys
      KERNEL: /usr/lib/debug/lib/modules/2.6.32-431.5.1.el6.x86_64/vmlinux
    DUMPFILE: /var/crash/127.0.0.1-2014-03-26-12:24:39/vmcore  [PARTIAL DUMP]
        CPUS: 1
        DATE: Wed Mar 26 12:24:36 2014
      UPTIME: 00:01:32
    LOAD AVERAGE: 0.17, 0.09, 0.03
       TASKS: 159
    NODENAME: elserver1.abc.com
     RELEASE: 2.6.32-431.5.1.el6.x86_64
     VERSION: #1 SMP Fri Jan 10 14:46:43 EST 2014
     MACHINE: x86_64  (2132 Mhz)
      MEMORY: 4 GB
       PANIC: "Oops: 0002 [#1] SMP " (check log for details)







