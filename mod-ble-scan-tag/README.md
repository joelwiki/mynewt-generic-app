# BLE generic application modules Package Definition

This is the package containing the generic firmware module for BLE operation for tag scanning

The source files are located in the src/ directory.

Header files are located in include/ 

pkg.yml contains the base definition of the package.

Any questions?  Please refer to the documentation at 
http://mynewt.apache.org/ or ask questions on dev@mynewt.apache.org

mod-ble-scan-tag
-------

mod_ble_scan_tag [module id = 3]: scans for BLE beacons with the MSB of the major !=0, ie both enter/exit and count types.
It takes all the found ids, and creates the enter/exit list based on a list it keeps between scans, and the count of each type. These are added to the UL packets. If there is too much data for the UL, currently no action is taken to avoid losing data...

Maximum numbers of tags scanned:
 - type/count : 100 in zone at same time (all types)
 - enter/exit : 16 in zone at same time
 These values are configured in the syscfg.yml for the target so are hardcoded for the firmware image. (as they define static array sizes to avoid malloc). They can be increase but it is likely the max RAM will be reached (eg at build time or runtime)
Maximum numbers of enter/exit/type counts per UL:
 - enter : 16
 - exit : 16
 - type/count : 16 (types)
 These values are set by configuration and can be altered. Note that the actuel limits in any particular UL may be reduced due to other data...
 
Useful Config keys:
------------------
0501 : scan time in millisecs

