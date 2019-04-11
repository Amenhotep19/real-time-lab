# Affinity Tuning

## Introduction

On Linux Symmetric Multi-Processor systems, it is possible to tune processes and IRQs to run on specific cores. This allows to reduce the number of user and kernel tasks running on cores expected to run the real-time applications.

The cores available in the system can be inspected with:
```
root@intel-corei7-64:~# ls -l /sys/devices/system/cpu/
total 0
drwxr-xr-x 7 root root    0 Apr 12 15:20 cpu0
drwxr-xr-x 7 root root    0 Apr 12 15:20 cpu2
drwxr-xr-x 5 root root    0 Apr 13 02:06 cpufreq
drwxr-xr-x 2 root root    0 Apr 13 02:06 cpuidle
drwxr-xr-x 2 root root    0 Apr 13 02:06 hotplug
-r--r--r-- 1 root root 4096 Apr 13 02:06 isolated
-r--r--r-- 1 root root 4096 Apr 13 02:06 kernel_max
drwxr-xr-x 2 root root    0 Apr 13 02:06 microcode
-r--r--r-- 1 root root 4096 Apr 13 02:06 modalias
-r--r--r-- 1 root root 4096 Apr 13 02:06 nohz_full
-r--r--r-- 1 root root 4096 Apr 13 02:06 offline
-r--r--r-- 1 root root 4096 Apr 12 15:20 online
-r--r--r-- 1 root root 4096 Apr 13 02:06 possible
drwxr-xr-x 2 root root    0 Apr 13 02:06 power
-r--r--r-- 1 root root 4096 Apr 12 15:20 present
-rw-r--r-- 1 root root 4096 Apr 12 15:20 uevent
```

There are two cores in this system, named cpu0 and cpu2.


Without any extra configuration, the processes are distributed between both cores. Using ps to list the attribute ```PSR``` (the processor where the process is running) yields:

```
root@intel-corei7-64:~# ps -eTo psr,pid,tid,policy,rtprio,cmd
PSR   PID   TID POL RTPRIO CMD
  2     1     1 TS       - /sbin/init initrd=\initrd LABEL=boot processor.max_cstate=0 intel_idle.max_cstate=0 clocksource=tsc tsc=reliable nmi_watchdog=0 nosoftlockup intel_pstate=disable i915.disable_power_
  0     2     2 TS       - [kthreadd]
  0     3     3 TS       - [ksoftirqd/0]
  0     4     4 FF       1 [ktimersoftd/0]
  0     6     6 TS       - [kworker/0:0H]
  2     8     8 FF       1 [rcu_preempt]
  2     9     9 FF       1 [rcu_sched]
  0    10    10 FF       1 [rcub/0]
  0    11    11 FF       1 [rcuc/0]
  2    12    12 TS       - [kswork]
  0    13    13 FF      99 [posixcputmr/0]
  0    14    14 FF      99 [migration/0]
  0    15    15 TS       - [cpuhp/0]
  2    16    16 TS       - [cpuhp/2]
  2    17    17 FF      99 [migration/2]
  2    18    18 FF       1 [rcuc/2]
  2    19    19 FF       1 [ktimersoftd/2]
  2    20    20 TS       - [ksoftirqd/2]
  2    22    22 TS       - [kworker/2:0H]
  2    23    23 FF      99 [posixcputmr/2]
  2    24    24 TS       - [kdevtmpfs]
  2    25    25 TS       - [netns]
  2    27    27 TS       - [oom_reaper]
  2    28    28 TS       - [writeback]
  2    29    29 TS       - [kcompactd0]
  2    30    30 TS       - [crypto]
  2    31    31 TS       - [bioset]
  2    32    32 TS       - [kblockd]
  2    33    33 FF      50 [irq/9-acpi]
  2    34    34 TS       - [md]
  2    35    35 TS       - [watchdogd]
  2    36    36 TS       - [rpciod]
  2    37    37 TS       - [xprtiod]
  2    39    39 TS       - [kswapd0]
  2    40    40 TS       - [vmstat]
  2    41    41 TS       - [nfsiod]
  2    63    63 TS       - [kthrotld]
  2    66    66 TS       - [bioset]
  2    67    67 TS       - [bioset]
  0    68    68 TS       - [bioset]
  0    69    69 TS       - [bioset]
  0    70    70 TS       - [bioset]
  0    71    71 TS       - [bioset]
  0    72    72 TS       - [bioset]
  0    73    73 TS       - [bioset]
  0    74    74 TS       - [bioset]
  0    75    75 TS       - [bioset]
  0    76    76 TS       - [bioset]
  0    77    77 TS       - [bioset]
  0    78    78 TS       - [bioset]
  0    79    79 TS       - [bioset]
  0    80    80 TS       - [bioset]
  0    81    81 TS       - [bioset]
  2    83    83 TS       - [bioset]
  0    84    84 TS       - [bioset]
  0    85    85 TS       - [bioset]
  0    86    86 TS       - [bioset]
  2    87    87 TS       - [bioset]
  0    88    88 TS       - [bioset]
  0    89    89 TS       - [bioset]
  0    90    90 TS       - [bioset]
  2    92    92 FF      50 [irq/27-idma64.0]
  2    93    93 FF      50 [irq/27-i2c_desi]
  2    94    94 FF      50 [irq/28-idma64.1]
  2    95    95 FF      50 [irq/28-i2c_desi]
  0    96    96 FF      50 [irq/29-idma64.2]
  0    98    98 FF      50 [irq/29-i2c_desi]
  0   100   100 FF      50 [irq/30-idma64.3]
  2   101   101 FF      50 [irq/30-i2c_desi]
  0   102   102 FF      50 [irq/31-idma64.4]
  2   103   103 FF      50 [irq/31-i2c_desi]
  2   104   104 FF      50 [irq/32-idma64.5]
  0   106   106 FF      50 [irq/32-i2c_desi]
  0   107   107 FF      50 [irq/33-idma64.6]
  2   108   108 FF      50 [irq/33-i2c_desi]
  0   109   109 FF      50 [irq/34-idma64.7]
  2   110   110 FF      50 [irq/34-i2c_desi]
  0   111   111 FF      50 [irq/4-idma64.8]
  2   112   112 FF      50 [irq/5-idma64.9]
  0   113   113 FF      50 [irq/35-idma64.1]
  2   115   115 FF      50 [irq/37-idma64.1]
  0   117   117 TS       - [nvme]
  0   118   118 FF      50 [irq/365-xhci_hc]
  0   121   121 TS       - [scsi_eh_0]
  0   122   122 TS       - [scsi_tmf_0]
  0   123   123 TS       - [usb-storage]
  0   124   124 TS       - [dm_bufio_cache]
  0   125   125 FF      50 [irq/39-mmc0]
  2   126   126 FF      49 [irq/39-s-mmc0]
  0   127   127 FF      50 [irq/42-mmc1]
  2   128   128 FF      49 [irq/42-s-mmc1]
  0   129   129 TS       - [ipv6_addrconf]
  0   144   144 TS       - [bioset]
  0   175   175 TS       - [bioset]
  0   177   177 TS       - [mmcqd/0]
  0   182   182 TS       - [bioset]
  2   187   187 TS       - [mmcqd/0boot0]
  0   189   189 TS       - [bioset]
  0   191   191 TS       - [mmcqd/0boot1]
  0   193   193 TS       - [bioset]
  0   195   195 TS       - [mmcqd/0rpmb]
  2   332   332 TS       - [kworker/2:1H]
  0   342   342 TS       - [kworker/0:1H]
  0   407   407 TS       - [jbd2/mmcblk0p2-]
  0   409   409 TS       - [ext4-rsv-conver]
  2   429   429 TS       - [bioset]
  0   463   463 TS       - [loop0]
  0   466   466 TS       - [jbd2/loop0-8]
  0   467   467 TS       - [ext4-rsv-conver]
  2   491   491 TS       - /lib/systemd/systemd-journald
  2   525   525 TS       - /lib/systemd/systemd-udevd
  0   533   533 TS       - /lib/systemd/systemd-timesyncd
  0   533   548 TS       - /lib/systemd/systemd-timesyncd
  0   551   551 TS       - /usr/sbin/syslog-ng -F -p /var/run/syslogd.pid
  2   553   553 TS       - /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
  0   558   558 FF      50 [irq/366-mei_me]
  2   559   559 FF      50 [irq/8-rtc0]
  2   560   560 FF      50 [irq/35-pxa2xx-s]
  2   561   561 TS       - [spi1]
  0   563   563 FF      50 [irq/37-pxa2xx-s]
  0   564   564 TS       - [spi3]
  2   571   571 TS       - /usr/sbin/jhid -d
  2   572   572 FF      50 [irq/369-i915]
  0   625   625 TS       - /usr/sbin/connmand -n
  0   626   626 TS       - avahi-daemon: running [intel-corei7-64.local]
  2   629   629 TS       - /lib/systemd/systemd-logind
  2   631   631 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  0   631   819 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  2   658   658 TS       - /usr/sbin/ofonod -n
  0   712   712 TS       - /usr/sbin/acpid
  0   770   770 TS       - /usr/sbin/rpcbind
  0   781   781 TS       - avahi-daemon: chroot helper
  0   798   798 FF       1 [i915/signal:0]
  0   800   800 FF       1 [i915/signal:1]
  2   801   801 FF       1 [i915/signal:2]
  2   802   802 FF       1 [i915/signal:4]
  0   821   821 TS       - /usr/sbin/rpc.statd -F
  0   828   828 TS       - /sbin/agetty --noclear tty1 linux
  0   829   829 TS       - /sbin/agetty -8 -L ttyS0 115200 xterm
  0   830   830 TS       - /sbin/agetty -8 -L ttyS1 115200 xterm
  2   832   832 FF      50 [irq/367-enp2s0]
  2   834   834 TS       - /usr/sbin/wpa_supplicant -u
  2   835   835 FF      50 [irq/368-enp3s0]
  2   836   836 TS       - /usr/sbin/nmbd
  2   844   844 FF      50 [irq/4-serial]
  0   846   846 FF      50 [irq/5-serial]
  0   849   849 TS       - /usr/sbin/smbd
  2   850   850 TS       - /usr/sbin/smbd
  2   851   851 TS       - /usr/sbin/smbd
  0   853   853 TS       - /usr/sbin/smbd
  2   855   855 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
  2   856   856 TS       - -sh
  0  3804  3804 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
  0  3805  3805 TS       - -sh
  0 12953 12953 TS       - [kworker/0:1]
  0 13034 13034 TS       - [kworker/u8:0]
  2 13041 13041 TS       - [kworker/2:2]
  2 13117 13117 TS       - [kworker/2:0]
  0 13161 13161 TS       - [kworker/u8:2]
  0 13190 13190 TS       - [kworker/0:0]
  2 13198 13198 TS       - [kworker/2:1]
  0 13266 13266 TS       - [kworker/0:2]
  2 13271 13271 TS       - /sbin/agetty -8 -L ttyS2 115200 xterm
  2 13272 13272 TS       - ps -eTo psr,pid,tid,policy,rtprio,cmd
```


## Critical isolation via kernel boot parameters

We are going to implement the following configuration:
* cpu2 (Critical Core): will run our real-time applications
* cpu0: will run everything else

Actually, cpu2 will also need to ru some operating system tasks, as Linux still owns the hardware.

We are going to use the following boot options, that accept a list of cores to act upon:

| Option | Parameter | Isolation |
| ------ | --------- | --------- |
| ```isolcpus``` | List of critical cores | The kernel scheduler will not migrate tasks from other cores to them |
| ```irqaffinity``` | List of non-critical cores | Protects the cores from IRQs |
| ```rcu_nocbs```   | List of critical cores | Stops RCU callbacks from getting called |
| ```nohz_full```   | List of critical cores | If the core is idle or has a single running task, it will not get scheduling clock ticks. Use together with nohz=off so dynamic ticks do not impact latencies. |

In our case, the resulting parameters will look like:
```isolcpus=2 irqaffinity=0 rcu_nocbs=2 nohz=off nohz_full=2```

Let us edit the following file and add those boot parameters:
```root@intel-corei7-64:~# vi /media/realroot/loader/entries/boot.conf```

The modified file should look like:
```
title boot
linux /vmlinuz
initrd /initrd
options LABEL=boot isolcpus=2 irqaffinity=0 rcu_nocbs=2 nohz=off nohz_full=2 processor.max_cstate=0            intel_idle.max_cstate=0            clocksource=tsc            tsc=reliable            nmi_watchdog=0            nosoftlockup            intel_pstate=disable            i915.disable_power_well=0            i915.enable_rc6=0            idle=poll            noht 3 snd_hda_intel.power_save=1 snd_hda_intel.power_save_controller=y scsi_mod.scan=async console=ttyS2,115200 rootwait console=ttyS0,115200 console=tty0
```

Now let us reboot. You will be logged off and will need to log in again once the system has completed booting again:
```
root@intel-corei7-64:~# reboot
Connection to 192.168.2.1 closed by remote host.
Connection to 192.168.2.1 closed.
```

## Checking the isolated configuration

Repeating the ```ps``` command we should be able to check if it worked:

```
root@intel-corei7-64:~# ps -eTo psr,pid,tid,policy,rtprio,cmd
PSR   PID   TID POL RTPRIO CMD
  0     1     1 TS       - /sbin/init initrd=\initrd LABEL=boot isolcpus=2 irqaffinity=0 rcu_nocbs=2 processor.max_cstate=0 intel_idle.max_cstate=0 clocksource=tsc tsc=reliable nmi_watchdog=0 nosoftlockup int
  0     2     2 TS       - [kthreadd]
  0     3     3 TS       - [ksoftirqd/0]
  0     4     4 FF       1 [ktimersoftd/0]
  0     6     6 TS       - [kworker/0:0H]
  0     8     8 FF       1 [rcu_preempt]
  0     9     9 FF       1 [rcu_sched]
  0    10    10 FF       1 [rcub/0]
  0    11    11 FF       1 [rcuc/0]
  0    12    12 TS       - [kswork]
  0    13    13 FF      99 [posixcputmr/0]
  0    14    14 FF      99 [migration/0]
  0    15    15 TS       - [cpuhp/0]
  2    16    16 TS       - [cpuhp/2]
  2    17    17 FF      99 [migration/2]
  2    18    18 FF       1 [rcuc/2]
  2    19    19 FF       1 [ktimersoftd/2]
  2    20    20 TS       - [ksoftirqd/2]
  2    21    21 TS       - [kworker/2:0]
  2    22    22 TS       - [kworker/2:0H]
  0    23    23 TS       - [rcuop/2]
  0    24    24 TS       - [rcuos/2]
  2    25    25 FF      99 [posixcputmr/2]
  0    26    26 TS       - [kdevtmpfs]
  0    27    27 TS       - [netns]
  0    29    29 TS       - [oom_reaper]
  0    30    30 TS       - [writeback]
  0    31    31 TS       - [kcompactd0]
  0    32    32 TS       - [crypto]
  0    33    33 TS       - [bioset]
  0    34    34 TS       - [kblockd]
  0    35    35 FF      50 [irq/9-acpi]
  0    36    36 TS       - [md]
  0    37    37 TS       - [watchdogd]
  0    38    38 TS       - [rpciod]
  0    39    39 TS       - [xprtiod]
  0    41    41 TS       - [kswapd0]
  0    42    42 TS       - [vmstat]
  0    43    43 TS       - [nfsiod]
  0    65    65 TS       - [kthrotld]
  0    68    68 TS       - [bioset]
  0    69    69 TS       - [bioset]
  0    70    70 TS       - [bioset]
  0    71    71 TS       - [bioset]
  0    72    72 TS       - [bioset]
  0    73    73 TS       - [bioset]
  0    74    74 TS       - [bioset]
  0    75    75 TS       - [bioset]
  0    76    76 TS       - [bioset]
  0    77    77 TS       - [bioset]
  0    78    78 TS       - [bioset]
  0    79    79 TS       - [bioset]
  0    80    80 TS       - [bioset]
  0    81    81 TS       - [bioset]
  0    82    82 TS       - [bioset]
  0    83    83 TS       - [bioset]
  0    84    84 TS       - [bioset]
  0    85    85 TS       - [bioset]
  0    86    86 TS       - [bioset]
  0    87    87 TS       - [bioset]
  0    88    88 TS       - [bioset]
  0    89    89 TS       - [bioset]
  0    90    90 TS       - [bioset]
  0    91    91 TS       - [bioset]
  0    93    93 FF      50 [irq/27-idma64.0]
  0    94    94 FF      50 [irq/27-i2c_desi]
  0    95    95 FF      50 [irq/28-idma64.1]
  0    96    96 FF      50 [irq/28-i2c_desi]
  0    97    97 FF      50 [irq/29-idma64.2]
  0    98    98 FF      50 [irq/29-i2c_desi]
  0    99    99 FF      50 [irq/30-idma64.3]
  0   100   100 TS       - [kworker/0:3]
  0   101   101 FF      50 [irq/30-i2c_desi]
  0   102   102 FF      50 [irq/31-idma64.4]
  0   103   103 FF      50 [irq/31-i2c_desi]
  0   104   104 FF      50 [irq/32-idma64.5]
  0   105   105 FF      50 [irq/32-i2c_desi]
  0   106   106 FF      50 [irq/33-idma64.6]
  0   108   108 FF      50 [irq/33-i2c_desi]
  0   109   109 FF      50 [irq/34-idma64.7]
  0   110   110 FF      50 [irq/34-i2c_desi]
  0   111   111 FF      50 [irq/4-idma64.8]
  0   112   112 FF      50 [irq/5-idma64.9]
  0   114   114 FF      50 [irq/35-idma64.1]
  0   115   115 FF      50 [irq/37-idma64.1]
  0   116   116 TS       - [nvme]
  0   118   118 FF      50 [irq/365-xhci_hc]
  0   119   119 TS       - [scsi_eh_0]
  0   120   120 TS       - [scsi_tmf_0]
  0   121   121 TS       - [usb-storage]
  0   122   122 TS       - [dm_bufio_cache]
  0   123   123 FF      50 [irq/39-mmc0]
  0   124   124 FF      49 [irq/39-s-mmc0]
  0   125   125 FF      50 [irq/42-mmc1]
  0   126   126 FF      49 [irq/42-s-mmc1]
  0   127   127 TS       - [ipv6_addrconf]
  0   142   142 TS       - [bioset]
  0   164   164 TS       - [bioset]
  0   165   165 TS       - [mmcqd/0]
  0   166   166 TS       - [bioset]
  0   167   167 TS       - [mmcqd/0boot0]
  0   168   168 TS       - [bioset]
  0   169   169 TS       - [mmcqd/0boot1]
  0   170   170 TS       - [bioset]
  0   171   171 TS       - [mmcqd/0rpmb]
  0   177   177 TS       - [kworker/0:1H]
  0   211   211 TS       - [bioset]
  0   265   265 TS       - [kworker/u8:2]
  0   392   392 TS       - [jbd2/mmcblk0p2-]
  0   395   395 TS       - [ext4-rsv-conver]
  0   533   533 TS       - [loop0]
  0   536   536 TS       - [jbd2/loop0-8]
  0   537   537 TS       - [ext4-rsv-conver]
  0   567   567 TS       - /lib/systemd/systemd-journald
  0   591   591 TS       - /lib/systemd/systemd-udevd
  0   613   613 TS       - /lib/systemd/systemd-timesyncd
  0   613   626 TS       - /lib/systemd/systemd-timesyncd
  0   628   628 TS       - /lib/systemd/systemd-logind
  0   630   630 TS       - /usr/sbin/syslog-ng -F -p /var/run/syslogd.pid
  0   631   631 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  0   631   739 TS       - /usr/sbin/thermald --no-daemon --dbus-enable
  0   634   634 TS       - avahi-daemon: running [intel-corei7-64.local]
  0   635   635 TS       - /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation
  0   638   638 FF      50 [irq/366-mei_me]
  0   640   640 FF      50 [irq/8-rtc0]
  0   644   644 FF      50 [irq/35-pxa2xx-s]
  0   645   645 TS       - [spi1]
  0   646   646 FF      50 [irq/37-pxa2xx-s]
  0   647   647 TS       - [spi3]
  0   648   648 TS       - /usr/sbin/rpcbind
  0   649   649 TS       - avahi-daemon: chroot helper
  0   650   650 FF      50 [irq/369-i915]
  0   654   654 TS       - /usr/sbin/jhid -d
  0   694   694 TS       - /usr/sbin/connmand -n
  0   696   696 TS       - /usr/sbin/ofonod -n
  0   705   705 TS       - /usr/sbin/acpid
  2   733   733 TS       - [kworker/2:1]
  0   800   800 FF       1 [i915/signal:0]
  0   808   808 TS       - /usr/sbin/rpc.statd -F
  0   810   810 FF       1 [i915/signal:1]
  0   819   819 FF       1 [i915/signal:2]
  0   825   825 FF       1 [i915/signal:4]
  0   882   882 TS       - /sbin/agetty -8 -L ttyS0 115200 xterm
  0   884   884 TS       - /sbin/agetty -8 -L ttyS1 115200 xterm
  0   886   886 TS       - /sbin/agetty --noclear tty1 linux
  0   905   905 TS       - /usr/sbin/wpa_supplicant -u
  0   906   906 FF      50 [irq/367-enp2s0]
  0   912   912 FF      50 [irq/368-enp3s0]
  0   914   914 TS       - /usr/sbin/nmbd
  0   922   922 FF      50 [irq/5-serial]
  0   924   924 FF      50 [irq/4-serial]
  0   925   925 TS       - /usr/sbin/dropbear -i -r /etc/dropbear/dropbear_rsa_host_key -B
  0   926   926 TS       - -sh
  0   931   931 TS       - /usr/sbin/smbd
  0   932   932 TS       - /usr/sbin/smbd
  0   933   933 TS       - /usr/sbin/smbd
  0   935   935 TS       - /usr/sbin/smbd
  2   953   953 TS       - [kworker/2:1H]
  0  1042  1042 TS       - [kworker/0:0]
  0  1552  1552 TS       - [kworker/u8:0]
  0  2373  2373 TS       - [kworker/0:2]
  0  2454  2454 TS       - [kworker/u8:1]
  0  2459  2459 TS       - [kworker/0:1]
  0  2522  2522 TS       - /sbin/agetty -8 -L ttyS2 115200 xterm
  0  2523  2523 TS       - ps -eTo psr,pid,tid,policy,rtprio,cmd
```

Now there are only a few kernel threads running on core2:
```
  2    25    25 FF      99 [posixcputmr/2]
  2    17    17 FF      99 [migration/2]
  2    18    18 FF       1 [rcuc/2]
  2    19    19 FF       1 [ktimersoftd/2]
  2    16    16 TS       - [cpuhp/2]
  2    20    20 TS       - [ksoftirqd/2]
  2    21    21 TS       - [kworker/2:0]
  2    22    22 TS       - [kworker/2:0H]
  2   733   733 TS       - [kworker/2:1]
  2   953   953 TS       - [kworker/2:1H]
```

We are going to use ```taskset``` to run cyclictest on specific cores and compare the results. For example:

```
taskset --cpu-list 0 command # Runs command on core0
taskset --cpu-list 2 command # Runs command on core2
```

Applying it to ```cyclictest```:

```
root@intel-corei7-64:~# taskset --cpu-list 0 cyclictest --loops=10000 --policy=fifo --priority=99
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.36 1.83 9.59 1/167 7165

T: 0 ( 7163) P:99 I:1000 C:  10000 Min:     14 Act:   57 Avg:   55 Max:     131
```

```
root@intel-corei7-64:~# taskset --cpu-list 2 cyclictest --loops=10000 --policy=fifo --priority=99
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.28 1.74 9.43 1/167 7169

T: 0 ( 7167) P:99 I:1000 C:  10000 Min:     14 Act:   44 Avg:   42 Max:      67
```

As we can see, after isolating the critical core, cyclictest yields better results on it.
