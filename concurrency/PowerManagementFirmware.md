# Power Management Firmware Tuning

Follow the steps below to tune power management configuration on an UP Squared board:

1. Switch on the laptop and pres F12 as the firmware initializes
2. Goto: Performance ->
    * Intel(r) SpeedStep(TM) -> **Disable**: This option enables or fisables the Intel(r) SpeedStep(TM) mode of the processor. Disabling this will put the processors into the highest performance state with no dynamic adjustment.
    * C-States Control -> **Disable**: This option allows the operating system to use additional power saving when idle
    * Intel(r) TurboBoost(TM) -> **Disable** Disabling this option disallows the TurboBoost(TM) driver from increasing the performance state of the processor above the standard performance.

3. Goto: Power Management -> Primary Battery Charge Configuration Standard -> Disables adaptive battery charging
