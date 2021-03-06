# Generic application package

This is the package containing the generic application core.

The source files are located in the src/ directory.

Header files are located in include/ 

pkg.yml contains the base definition of the package.

Any questions?  Please refer to the documentation at 
http://mynewt.apache.org/ or ask questions on dev@mynewt.apache.org
/**
 * Copyright 2019 Wyres
 * Licensed under the Apache License, Version 2.0 (the "License"); 
 * you may not use this file except in compliance with the License. 
 * You may obtain a copy of the License at
 *    http://www.apache.org/licenses/LICENSE-2.0
 * Unless required by applicable law or agreed to in writing, 
 * software distributed under the License is distributed on 
 * an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, 
 * either express or implied. See the License for the specific 
 * language governing permissions and limitations under the License.
*/

The app-core framework codifies a standard state machine for the code operation of a lora based device.
It has the 'app-core' control, and 1 or more 'mod-X' packages which define independant modules,
which will be called at the appropriate points to collect their data ready for UL.

The 'app-core' package governs the central execution. It starts with an initialisation phase. 
This uses the mynewt sysinit mechanisme, to allow the 'modules' to be initialised
based on their pkg.yml sysinit values, and to register with the app-core system as both modules to be
executed and as the destination of DL actions. Registeration of a module essentially consists of it indicating the set of 
functions that implement the API required.
Once sysinit has finished, app-core state machine is run, beginning in the 'startup' state.
State machine:
STARTUP: activates the console for a specific period of time for config AT commands. Once expired, the SM checks if all 'critical' config was found in the PROM (devEUI/appKey currently). If so, it goes to the TRYJOIN phase, else it immediately enters STOCK mode.
TRY-JOIN / RETRY-JOIN:
The appcore asks the lorawan api to JOIN the network. If the join attempt fails, then either the code goes into STOCK mode (if it has NEVER joined) , or into RETRY-JOIN, to sleep for X minutes before retrying the JOIN. If the JOIN is sucessful then stock mode is disabled, and transition to GETTING-SERIAL.
STOCK:
This consists of deep sleep until manually reset and is intended for devices held in stock, either because they have no valid config, or because the local lorawan network is not configured to let them JOIN.
GETTING-SERIAL: 
Data collection from modules that must be executed in 'serial' mode ie when no other module is also executing.
This lets a module be sure its use of hardware specific elements is not in competition with other modules. The execution time 
each module requires is returned from its start() method, although a module can indicate an earlier termination at any time.
GETTING-PARALLEL: 
Data collection from modules that can execute in parallel. This lasts as long as the longest module timeout.
SENDING-UL: 
Tx of the lorawan UL : the collected data in 1 or more messages is sent as UL messages. Any DL packet received is decoded and the actions within are interpreted.
IDLE: idleness : the machine sleeps globally for the configured amount of time. It actually wakes every 60s and checks for movement as the idle time may be set differently for the moving/not moving cases.
A module may also register a 'tic' hook ie a function which is called during these regular wakeups in IDLE to perform an action.
During IDLE the lowpower mode requested is DEEPSLEEP to achieve the lowest current consumation.

LoRa Operation
--------------
The Lorawan api used is defined in the generic/loraapi_XXX packages. The same lorapi.h API is used no matter which stack underneath.
generic/loraapi_KLK:
This package uses the KLK wrapper round the stackforce stack. 
The JOIN phase is explicity handled by the appcore statemachine. Note that the KLK wrapper auto-joins on 1st UL, the lorawan api forces this to happen when join is requested by sending an initial UL packet with the current uptime.
generic/loraapi_SKF:
This package wraps directly onto the stackforce API. Note : not yet complete.

An UL is not neccessarily sent every data collection loop - each module indicates if it has 'critical' data in its collection, and if no module is critical then no UL is sent. The config key MAXTIME_UL_MIN (0405) sets the maximum time that can elapse without an uplink (default 120 minutes) after which the UL data is sent anyway.

AT Command console
-------------------
The AppCore console is activated for all build profiles for 30s post-boot on the standard UART interface. If no 'AT' command
is received during this time, it transitions to the usual operation. If an AT command is received, the code stays permanently in the console, and the user must explicitly exit either via a reboot (ATZ) or a run (AT+RUN).

Useful AT commands:
- AT - wake the console
- AT+HELP - list the available commands
- ATZ - reboot
- AT+RUN - execute the data collection loop
- AT+INFO - some basic card info
- AT+GETCFG <config group> - show config keys for this group
- AT+SETCFG <4 digit key> <value> - set a config value
- AT+GETMODS/AT+SETMODS - see/change the set of activated modules. See app_core.h for the module ids.

AppCore module config keys
---------------------------
See app_core.h for the list. Some key ones:
0101 : devEUI - a critical config value - if not set then the appcore remains in STOCK mode
0103 : appKey - also a critical config value.
0401/0402 : idle time when moving (in seconds) / not moving (in minutes)
0407 : idle period check time (in seconds, 60s default)
0408 : stock mode : 0 = goto stock mode if JOIN fails, 1=retry if JOIN fails

| module    | config ID | length |                                          description  
| --------: | :-------: | :----: | :---------------------------------------------------------------------------------------: 
| UTIL      | 0001      | 8      | Reboot reason 
| UTIL      | 0002      | 64     | store fn tracker buffer in case of reboot 
| UTIL      | 0003      | 4      | Assert caller 
| UTIL      | 0004      | 1      | Accelerometer detection mode (0 = OFF, 1 = SHOCK DETECTION, 2 = FREE FALL DETECTION) 
| UTIL      | 0005      | -      | Shock detection threshold 
| UTIL      | 0006      | -      | Free fall detection threshold 
| UTIL      | 0007      | -      | Minimal shock detection duration (in tenth of seconds, default value = 6) 
| UTIL      | 0008      | -      | Minimal free fall detection duration (in tenth of seconds, default value = 6) 
| LORA      | 0101      | -      | Dev EUI 
| LORA      | 0102      | -      | App EUI 
| LORA      | 0103      | -      | App KEY 
| LORA      | 0104      | -      | Dev ADDR 
| LORA      | 0105      | -      | Network session key 
| LORA      | 0106      | -      | App session key 
| LORA      | 0107      | -      | ADR enabled 
| LORA      | 0108      | -      | Acknoledgement 
| LORA      | 0109      | -      | data rate 
| LORA      | 010A      | -      | tx power 
| LORA      | 010B      | -      | port for TX 
| LORA      | 010C      | -      | port for RX 
| APP       | 0201      | -      | CFG_UTIL_KEY_CHECKINTERVAL repos\generic\generic\include\wyres-generic\appConfigKeys.h  used ? 
| WYRES     | 0301      | -      | CFG_WYRES_KEY_TAG_SYNC_DM_INTERVAL  used ? 
| WYRES     | 0302      | -      | CFG_WYRES_KEY_LORA_TXPOWER  used ? 
| WYRES     | 0303      | -      | CFG_WYRES_KEY_BLE_SCANWINDOW  used ? 
| WYRES     | 0304      | -      | CFG_WYRES_KEY_TAG_NONSYNC_DM_INTERVAL  used ? 
| WYRES     | 0305      | -      | CFG_WYRES_KEY_FOTA_VER  used ? 
| APP_CORE  | 0401      | -      | idle time when moving (in seconds) 
| APP_CORE  | 0402      | -      | idle time not moving (in minutes) 
| APP_CORE  | 0403      | -      | MODSETUP_TIME_SECS 
| APP_CORE  | 0404      | -      | MODS_ACTIVE_MASK 
| APP_CORE  | 0405      | -      | Maximal time between uplink in minutes 
| APP_CORE  | 0406      | -      | Downlink id (dlid) 
| APP_CORE  | 0407      | -      | idle period check time (in seconds, 60s default) 
| APP_CORE  | 0408      | -      | Stock mode 
| APP_CORE  | 0409      | -      | Join timeout (in seconds) 
| APP_CORE  | 040A      | -      | Join retry interval (in minutes) 
| APP_CORE  | 040B      | -      | Firmware infos 
| APP_MOD   | 0501      | -      | BLE scan duration un ms 
| APP_MOD   | 0502      | -      | GPS cold time in seconds 
| APP_MOD   | 0503      | -      | GPS warm time in seconds 
| APP_MOD   | 0504      | -      | GPS power mode 
| APP_MOD   | 0505      | -      | GPS fix mode 
| APP_MOD   | 050A      | -      | Max navigation BLE per uplink 
| APP_MOD   | 050B      | -      | Ble EXIT timeout (in minuts) 
| APP_MOD   | 050C      | -      | Max enter per uplink 
| APP_MOD   | 050D      | -      | Max exit per uplink 
| APP_MOD   | 0510      | -      | iBeacon UUID 
| APP_MOD   | 0511      | -      | iBeacon major 
| APP_MOD   | 0512      | -      | iBeacon minor 
| APP_MOD   | 0513      | -      | iBeacon period (in milisecond) 
| APP_MOD   | 0514      | -      | iBeacon txPower 
| APP_MOD   | 0520      | -      | Pressure reference 
| APP_MOD   | 0521      | -      | Pressure offset 
     
DL Action handling      
------------------
App-core handles the reception and decoding of the DL packets. These consist of a set of 'actions', each with a 1 byte key (defined in app_core.h). Modules can register to execute specific action keys at startup - only 1 module can register for each key and the system will assert() if more than one tries.
Most of the core actions are handled by the app_core.c file, including reset, get/setcfg and setting UTCTime.


modules :
------------------

| KEY | ID |
| --------: | :--------: |
| APP_MOD_ENV | 0 |
| APP_MOD_GPS | 1 |
| APP_MOD_BLE_SCAN_NAV | 2 |
| APP_MOD_BLE_SCAN_TAGS | 3 |
| APP_MOD_BLE_IB | 4 |
| APP_MOD_IO | 5 |
| APP_MOD_PTI | 6 |


UL keys :
-------------------

| KEY | ID (decimal) | Description |
| --------: | :--------: | :--------: |
| APP_CORE_UL_VERSION | 0 | |
| APP_CORE_UL_UPTIME | 1 | |
| APP_CORE_UL_CONFIG | 2 | |
| APP_CORE_UL_ENV_TEMP | 3 | |
| APP_CORE_UL_ENV_PRESSURE | 4 | |
| APP_CORE_UL_ENV_HUMIDIT | 5 | |
| APP_CORE_UL_ENV_LIGHT | 6 | |
| APP_CORE_UL_ENV_BATTERY | 7 | |
| APP_CORE_UL_ENV_ADC1 | 8 | |
| APP_CORE_UL_ENV_ADC2 | 9 | |
| APP_CORE_UL_ENV_NOISE | 10 | |
| APP_CORE_UL_ENV_BUTTON | 11 | |
| APP_CORE_UL_ENV_MOVE | 12 | |
| APP_CORE_UL_ENV_FALL | 13 | |
| APP_CORE_UL_ENV_SHOCK | 14 | |
| APP_CORE_UL_ENV_ORIENT | 15 | |
| APP_CORE_UL_ENV_REBOOT | 16 | |
| APP_CORE_UL_ENV_LASTASSERT | 17 | |
| APP_CORE_UL_BLE_CURR | 18 | |
| APP_CORE_UL_BLE_ENTER | 19 | |
| APP_CORE_UL_BLE_EXIT | 20 | |
| APP_CORE_UL_BLE_COUNT | 21 | |
| APP_CORE_UL_GPS | 22 | |
| APP_CORE_UL_BLE_ERRORMASK | 23 | |

DL keys : 
-------------------------
| KEY | ID (decimal) | Description |
| --------: | :--------: | :--------: |
| APP_CORE_DL_REBOOT | 1 | - |
| APP_CORE_DL_SET_CONFIG | 2 | - |
| APP_CORE_DL_GET_CONFIG | 3 | - |
| APP_CORE_DL_FLASH_LED1 | 5 | - |
| APP_CORE_DL_FLASH_LED2 | 6 | - |
| APP_CORE_DL_SET_UTCTIME | 24 | - |
| APP_CORE_DL_FOTA | 25 | - |
| APP_CORE_DL_GET_MODS |26 | - |
| APP_CORE_DL_FIX_GPS | 11 | - |