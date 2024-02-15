<!--
 * @Author: ircama
 * @Date: 2020-03-31 19:46:54
 * @LastEditTime: 2024-02-10 16:53:37
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: /Telink_825X_SDK/example/at/README.md
 -->
# BLE-ADV-TELINK Project

This code is based on [Telink_825X_SDK](https://github.com/Ai-Thinker-Open/Telink_825X_SDK) and is tested on a [Ai-Thinker TB-03F-KIT](https://docs.ai-thinker.com/_media/tb-03f-kit_specification_en.pdf).

It reuses the [AT example](https://github.com/Ai-Thinker-Open/Telink_825X_SDK/tree/master/example/at) included in the SDK, modifying it in order to appropriately process BLE advertisements for a FRITZ!Box device.

[TLSR8253 chip](http://wiki.telink-semi.cn/doc/ds/DS_TLSR8253-E_Datasheet%20for%20Telink%20BLE+IEEE802.15.4%20Multi-Standard%20Wireless%20SoC%20TLSR8253.pdf) documentation: http://wiki.telink-semi.cn/wiki/chip-series/TLSR825x-Series/

The code offers USB serial UART interface at 115200 bps, N, 8, 1 and uses DTR and RTS. To reset the device and start serial communication:

- open the serial port,
- set DTR on,
- set RTS on,
- wait 50 millisecs,
- set DTR off,
- set RTS off.

Changes vs. the original firmware:

- added `AT+SCAN` setting modes, including continuous advertising in passive mode, which can also be activated at boot
- advertising duplicates are not disabled
- using the colored LED to show the device brand of each received advertising
- added filter advertising on specific three brands

To enable scanning and to set `AT+SCAN=<value>`, the device shall be in host/master mode (`AT+MODE=1`):

Setting|Device mode
---|---
`AT+MODE=0`|Slave
`AT+MODE=1`|Master
`AT+MODE=2`|iBeacon

The following SCAN modes are allowed:

|AT setting code|Return code|Note|
|---|---|---|
|`AT+SCAN`|Output depends on the SCAN mode|Start SCAN without modifying the SCAN mode|
|`AT+SCAN=0`|`+SCAN_TYPE:0`, |Start SCAN and automatically disable it after 3 seconds (default)|
|`AT+SCAN=1`|`+SCAN_TYPE:1`, `+SCAN_SET_CONTINUOUS`|Start SCAN with no timeout|
|`AT+SCAN=2`|`+SCAN_TYPE:2`, `+SCAN_SET_AUTO`|The SCAN is automatically started at boot, with no need of `AT+SCAN=2`
|`AT+SCAN=3`|`+SCAN_TYPE:3`, `+SCAN_SET_AUTO_FILTER`|Same as `AT+SCAN=2`, but the scan filters only the brands in the following table

Recognized MAC brands:

Brand MAC|Brand name|IC PIN (LED)
--------|---|---
A4:C1:38|Telink Semiconductor (Taipei) Co. Ltd|GPIO_PC2
54:EF:44|Lumi United Technology Co., Ltd|GPIO_PC3
E4:AA:EC|Tianjin Hualai Tech Co, Ltd|GPIO_PC4
Any other MAC|filtered out with AT+SCAN=3|no led

Change the code *app_master.c* to edit the trailing parts of MAC addresses which are filtered and to control related LED color.

All settings (e.g., `AT+MODE=<n>` and `AT+SCAN=<n>`) are permanently stored into the non-volatile RAM of the device.

# Compiling and installing

- Copy the folder on a Linux system (Ubuntu). WSL is supported.
- `cd ble-adv-telink`
- `make`

The produced firmware is `src/out/ble-adv-telink.bin`.

To burn the firmware with a PC, connect the device via USB and use the software [Ai-Thinker_TB_Tools_V1.5.0.exe](https://ai-thinker.oss-cn-shenzhen.aliyuncs.com/TB_Tool/Ai-Thinker_TB_Tools_V1.5.0.exe) and check [related repository](https://github.com/Ai-Thinker-Open/TBXX_Flash_Tool/tree/1.x.x). The same software can be used to test the AT commands.

- Select the appropriate COM port
- press the button with "..." and select the firmware
- press the right side button to the previously mentioned one "..."

To enter an AT command, use the second tab. Select the port and bitrate, press the second button to connect the device, verify that the checkbox is selected.

-----------------------------------------

# AT firmware design principle - Original description (English translation from Chinese)

The following comments come from the original code (AT version 0.7.4) and might be now partially superseded by the modifications.

## AT Design Principles

Added comments to some files

## AT mode selection and conversion

If the control is not processed (left floating) and the module is not connected to the mobile phone, it will be in AT mode and can respond to AT commands. After the module is connected to the mobile phone, it enters the transparent transmission mode. In the transparent transmission mode, the data sent by the MCU to the module through the serial port will be forwarded intact to the mobile phone via Bluetooth by the module. Similarly, the data sent by the mobile phone to the module through Bluetooth will be transmitted intact to the MCU through the serial port.


|   Module|Serial port TX|Serial port RX|Control pin|Low power consumption status indication pin|Connection status indication pin|
|---------|--------------|--------------|-----------|-------------------------------------------|--------------------------------|
|TB-01    |PB1           |PB0           |PC5        |None                                       |None                            |
|TB-02+   |PB1           |PA0           |PC5        |PC3                                        |PC4                             |
|TB-02_Kit|PB1           |PB7           |PC5        |PC3                                        |PC4                             |


When the module is not connected to the mobile phone, it will be in AT mode and can respond to AT commands. After connecting with the mobile phone, it will enter the transparent transmission mode and no longer respond to AT commands. If the user needs to send AT commands in transparent transmission mode, the control pin can be pulled low. After pulling low, the module will temporarily enter AT mode, and return to transparent transmission mode after releasing it. The status corresponds to the following table:

||No connection established with mobile phone|Connection established with mobile phone
|---|---|---|
|CONTROL_GPIO is high level|AT mode|Transparent transmission mode
|CONTROL_GPIO is low |AT mode |AT mode
||STATE pin is low|STATE pin is high|

Note: If the user does not need to use transparent transmission mode, just pull down CONTROL_GPIO through a resistor. In AT mode, data can be sent through the AT+SEND command.

## Modify control and indication pins

The above `control pin` `low power status indication pin` `connection status indication pin` is defined in the `app_config.h` file, if necessary Can be modified by yourself.

## Serial port adaptation

The serial port configuration part is in the `app_uart.c` file. The default TX is PB1, and the RX pin adopts an adaptive method. The level status of PA0, PB0, and PB7 is detected after power-on. If one is high level , then set it as serial port Rx.


## AT command format
AT commands can be subdivided into four format types:

|Type|Command format|Description|Remarks
|---|---------|---|---|
|Query command|AT+?|Query the current value in the command.	
|Setting command|AT+
    =<…>|Set user-defined parameter values.	
|Execute command|AT+
     |Perform some function with immutable parameters.	
|Test command|AT+
      =?|Return command help information	


## AT command set

|Serial number|Command|Function|Remarks|
|----|-----|----|----|
|1|AT|Test AT|
|2|ATE|Switch echo|
|3|AT+GMR|Query firmware version|
|4|AT+RST|Restart module
|5|AT+SLEEP|Deep sleep|
|6|AT+ RESTORE|Restore factory settings|Will restart after recovery|
|7|AT+BAUD|Query or set the baud rate|It will take effect after restart|
|8|AT+NAME|Query or set the Bluetooth broadcast name|It will take effect after restart|
|9|AT+MAC|Set or query the module MAC address|It will take effect after restart|
|10|AT+MODE|Query or master-slave mode
|11|AT+STATE|Query Bluetooth connection status
|12|AT+SCAN|Initiate scan in host mode
|13|AT+CONNECT|Initiate connection in host mode
|14|AT+DISCON|Disconnect
|15|AT+SEND|Send data in AT mode|
|16|+DATA|Data received in AT mode|
|17|AT+ADVDATA|Set the manufacturer-defined field content in the broadcast data|
|18|AT+LSLEEP|Set or enter light sleep|
|19|AT+RFPWR|Set or read transmit power|
|20|AT+IBCNUUID|Set or read iBeacon UUID|
|21|AT+MAJOR|Set or read iBeacon Major|
|22|AT+MINOR|Set or read iBeacon Minor|

## Host mode
In master mode, the module can communicate with another slave module. The main operations are as follows:

Configure the module into host (master) mode:

	AT+MODE=1

Scan surrounding modules:

	AT+SCAN

Connect the modules specified by the guide:

	AT+CONNECT=AC04187852AD

Please replace the above MAC address with the MAC address of your slave module.

Returning `OK` means the connection is successful. Use the following command to send data to the slave:

	AT+SEND=5,12345

Note: There is only AT command mode in the host state, and there is no transparent transmission mode.

## Low power consumption
The AT firmware supports two sleep modes, namely `deep sleep` and `light sleep`. In deep sleep mode, except for the GPIO wake-up function, all other functions of the module are turned off, and the power consumption is 1uA. one time. In addition to retaining GPIO wake-up, the light sleep mode also maintains the Bluetooth function. The power consumption is determined by the broadcast parameters, with an average of less than 10uA.

Enter deep sleep mode:

	AT+SLEEP

After executing the appeal instruction module and returning OK, it will immediately enter sleep mode and set the serial port RX as the wake-up pin. Send any character to the module again to wake it up.

Light sleep settings:
In the disconnected state, send the following command and the module will enter light sleep mode:

	AT+LSLEEP

In light sleep mode, the module will still perform Bluetooth broadcast. Light sleep mode no longer responds to any AT commands, and any data can be sent through the serial port RX pin to wake up the module.

When another Bluetooth device is successfully connected to the module, the module will also be woken up.

Automatically enter light sleep mode after power on:

	AT+LSLEEP=1

It does not automatically enter light sleep mode after powering on;

	AT+LSLEEP=0

Note: Light sleep mode only works in the slave state. If low power consumption is used, it is not recommended to set the baud rate below 115200. If the baud rate is too low, sending data through the serial port will take up a lot of time, thus affecting power consumption.

## iBeacon Mode
iBeacon is a special set of broadcast formats defined by Apple, mainly used for indoor positioning.
This iBeacon broadcast packet is 30 bytes in total, and the data format is as follows:

	02 # The number of bytes of the first AD structure (the number of next bytes, here is 2 bytes)
	01 # Flag of AD type
	1A # Flag value 0x1A = 000011010  
	bit 0 (OFF) LE Limited Discoverable Mode
	bit 1 (ON) LE General Discoverable Mode
	bit 2 (OFF) BR/EDR Not Supported
	bit 3 (ON) Simultaneous LE and BR/EDR to Same Device Capable (controller)
	bit 4 (ON) Simultaneous LE and BR/EDR to Same Device Capable (Host)
	1A # The number of bytes of the second AD structure (the next number of bytes, here is 26)
	FF # AD type flag, here is Manufacturer specific data. More flags can be found on the BLE official website: for example, 0x16 represents servicedata
	4C 00 # Company logo (0x004C == Apple)
	02 # Byte 0 of iBeacon advertisement indicator
	15 # Byte 1 of iBeacon advertisement indicator
	B9 40 7F 30 F5 F8 46 6E AF F9 25 55 6B 57 FE 6D # iBeacon proximity uuid
	00 01# major
	00 01 #minor
	c5 # calibrated Tx Power


TB series modules support sending iBeacon broadcasts. In iBeacon mode, the module can send broadcasts according to iBeacon format. The main operations are as follows:

Configure the module in iBeacon mode:

	AT+MODE=2

Set the UUID of iBeacon (hexadecimal format, 16 bytes in total):

	AT+IBCNUUID=11223344556677889900AABBCCDDEEFF

Set the MAJOR of iBeacon (hexadecimal format, 2 bytes in total):

	AT+MAJOR=1234

Set the MINOR of iBeacon (hexadecimal format, 2 bytes in total):

	AT+MINOR=4567

Note: The above commands will take effect after restarting and will be saved after power off. In conjunction with setting the broadcast gap, automatic light sleep can reduce iBeacon power consumption.

## FAQ

- 1/ Modify the default name of Bluetooth broadcast, the default is "Ai-Thinker"

```
//app.c
const u8 tbl_scanRsp [] = {
    0x0B, 0x09, 'A', 'i', '-', 'T', 'h', 'i', 'n', 'k', 'e', 'r',
};
```