# Power Management Firmware Tuning

Follow the steps below to tune power management configuration on an UP Squared board:

1. Make sure a screen and a keyboard are attached to the board
1. Plug the Live USB Stick
1. Switch on the UP Squared board
1. Select boot into firmware console
![1](https://github.intel.com/storage/user/9705/files/890ad1ce-6eeb-11e8-9f6e-2ba089013f95)
1. Press ESC to get to the main menu
![2](https://github.intel.com/storage/user/9705/files/c157ccda-6eeb-11e8-9264-76a7965145bc)
1. Goto: CRB Chipset -> South Bridge <br/>
Southbridge controls I/O functions, like USB, audio, serial...
![3](https://github.intel.com/storage/user/9705/files/cd488e1c-6eeb-11e8-9518-48595c9e784d)
![4](https://github.intel.com/storage/user/9705/files/dae46320-6eeb-11e8-817f-dd836df2ca4a)
   * Set Real Time Option -> RT Enabled, Agent Disabled <br/>
   This option optimizes PCIe transactions to achieve real-time performance.
![5](https://github.intel.com/storage/user/9705/files/e79517d6-6eeb-11e8-8615-49e72c297ffc)
1. Goto: CRB Advanced -> CPU Configuration -> CPU Power Management
![6](https://github.intel.com/storage/user/9705/files/f261fc60-6eeb-11e8-91b0-e218c3eb857b)
![7](https://github.intel.com/storage/user/9705/files/fce203b0-6eeb-11e8-842a-a59857c63802)
![8](https://github.intel.com/storage/user/9705/files/07d513e8-6eec-11e8-8eb8-f44ca7dc5734)
   * EIST	\<Disabled\> <br/>
   Enhanced Intel SpeedStep Technology (EIST) is used to dynamically control the CPU clock speed as per the load demand. The CPU clock speed is throttled for workloads with less demand which will reduce the power consumption of the CPU. For real-time workloads, we may disable this feature in order to make latencies more predictable.
   * Turbo Mode	\<Disabled\> <br/>
   If enabled, the processor cores will run at increased clock speed. For real-time workloads, we may disable this feature in order to make latencies more predictable.
   * Boost Performance Mode	\<Max Performance\> <br/>
   This option will enable BIOS to set the performance mode before booting to OS.
   * Power Limit 1 Enable	\<Disabled\> <br/>
   This option is used to enable/disable power limit options.
1. Goto: CRB Chipset -> Uncore Configuration <br/>
Refers to the functions which are connected closely to the core for achieving high performance, but are not considered part of it, e.g. the memory controller or L3 cache. It may also be referred to as "System Agent".
   * RC6(Render Standby)  	\<Disabled\> <br/>
   Provide options to put onboard graphics on standby mode for decreasing power consumption.
![9](https://github.intel.com/storage/user/9705/files/145945c6-6eec-11e8-8259-945a216bca53)
   * GT PM Support	\<Disabled\> <br/>
   Enable/Disable power management features supported by the CPU.
![10](https://github.intel.com/storage/user/9705/files/1fbe1630-6eec-11e8-9cf1-ec41ddfe857b)
