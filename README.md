### Original Airbreak readme [here](README_ORIG.md).

### This fork is intended to:
- Adjust the patcher to work on the Aircurve 10 S ASV, hardware model 37162 and firmware version SX567-0401.  
- Remove all optional modification options aside from the graph.c replacement code to support firmware modification from a known working modified state.  
- Serve as documentation of progress towards the goal of automating daily session data exfiltration.  

### A makeshift modification/compilation/flashing/debugging envrionment is set up as follows:
- An [STLINK-V3MINIE](https://www.st.com/en/development-tools/stlink-v3minie.html) STM32 programmer/debugger.
    - This is connected to the debugging header as described at [airbreak/disassembly](https://airbreak.dev/disassembly/). As per the Warning on the page, the genuine STLINK needs ```STM32_VDD``` on the machine connected to ```T_VCC``` on the STLINK so the STLINK can detect the target voltage.
- The [OpenOCD](https://openocd.org/) project setup as described at [airbreak/firmware](https://airbreak.dev/firmware/). Namely, running this command as root from the project folder:
    ```Shell
    openocd -f interface/stlink-v2.cfg -f 'tcl/airsense.cfg'
    ```
    - When using the STLINK-V3MINIE, ```stlink-v2.cfg``` is replaced with ```stlink.cfg```
    - Both telnet and gdb are connected to OpenOCD via these commands in different terminal windows:  
    ```telnet localhost 4444```  
    ```gdb``` and then once in gbd's terminal, ```target extended-remote localhost:3333```  
    gdb ```halt```'s the target on connect by default, so the machine will stop working until the ```continue``` command is issued in gdb once that happens. ```Ctrl-C``` will ```halt``` the machine's cpu so breakpoints and other debugging can be set up.
- [Ghidra](https://github.com/NationalSecurityAgency/ghidra) for decompiling the binary firmware
    - The binary firmware dumped via the STLINK is imported into Ghidra, choosing Cortex M little endian as the language, and setting the Base address at ```0x08000000```.
    - Before letting it analyze the binary, add entries to the Memory Map.
        - SRAM starts at ```0x20000000``` and is ```0x20000``` in size.
        - Using the SVD-loader extension to load ```STM32F405.svd``` downloaded from an SVD repisitory will automate adding the rest of the relevent memory address space.
    - The Airbreak repository contains a Ghidra2stubs script and a stubs.S file, but does not include the original xml from Ghidra. The stubs.S file can be used to cross-reference functions seen in Ghidra (although they will be off by 1 due to Thumb code).

The Debugger is connected to a virtual machine running Ubuntu Server with SSH access.
There are 2 interfaces exposed by OpenOCD that are used, the GDB debugging interface on port 3333 and a telnet interface for running commands on port 4444.  
One advantage of the telnet interface is the ability to dump memory from the target while it is stays running. Knowing the needed data most likely sits in RAM before being written to the SD card, dumps are taken of the entire 128KB ram address space at various points during a test run using this command while redirecting telnet stdout to a file:  
```mdd 0x20000000 8000```  
This will output 8000 64-bit doublewords (128KB). Output is formatted so it can be converted from hex to ascii to search for human-readable strings.  

Starting at address ```0x2000d5b0```, data that looks similar to what is found within the EDF files on the sd card can be seen occasionally during a session. A watchpoint is set on that address in 