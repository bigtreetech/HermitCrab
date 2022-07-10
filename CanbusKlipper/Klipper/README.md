# How to use Klipper on HermitCrab Canbus version

## Pinout
### HermitCrab Canbus pinout
<img src=Images/canbus_pinout.jpg width="400" /><br/>

## Wiring diagram
### Raspberry pi communicates with the hotend board via USB
<img src=Images/wiring_usb.png width="1200" /><br/>

### Raspberry pi communicates with the hotend board via Canbus
<img src=Images/wiring_canbus.png width="1200" /><br/>

## Generate the firmware.bin

1. Precompiled firmware
   * [firmware-F072-USB.bin](./firmware-F072-USB.bin) is the firmware that communicates with raspberry pi through USB
   * [firmware-F072-Canbus.bin](./firmware-F072-Canbus.bin) is the firmware that communicates with raspberry pi through Canbus

2. Build your own firmware
   1. Refer to [klipper's official installation](https://www.klipper3d.org/Installation.html) to download klipper source code to raspberry pi
   2. `Building the micro-controller` with the configuration shown below.
      * [*] Enable extra low-level configuration options
      * Micro-controller Architecture = `STMicroelectronics STM32`
      * Processor model = `STM32F072`(Note: You can set `Processor model` to `STM32F070` or wait for [PR](https://github.com/Klipper3d/klipper/pull/4799) to merge into the master branch and use the latest version of klipper firmware if your firmware has no USB option for `STM32F072`)
      * Clock Reference = `8 MHz crystal)`
      * IF USE USB
         * Communication interface = `USB (on PA11/PA12)`
         * USB ids `0x1d50` for USB vender ID, and `0x614f` for USB device ID. (Note: it is important that hte USB device ID is not the default `0x614e` assigned by Klipper as this causes the device to lose connection during `FIRMWARE_RESTART`
      * ElSE IF USE CANBUS
         * Communication interface = `CAN bus (on PB8/PB9)`
         * CAN bus speed = 250000 ([firmware-F072-Canbus.bin](./firmware-F072-Canbus.bin) default speed is 250K, You can set it by yourself, But you need to reduce the speed if the communication fails)

      <img src=Images/menuconfig.png width="800" /><br/>
    3. The `klipper.bin` file will be generated in the folder `home/pi/kliiper/out` after `make`. And you can use the windows computer under the same LAN as raspberry pi to copy `klipper.bin` from raspberry pi to the computer with `pscp` command in the CMD terminal. such as `pscp -C pi@192.168.0.101:/home/pi/klipper/out/klipper.bin c:\klipper.bin`(The terminal may prompt that `The server's host key is not cached` and ask `Store key in cache?((y/n)`, Please type `y` to store. And then it will ask for a password, please type the default password `raspberry` for raspberry pi)

## Update the firmware.bin
1. Connect the USB of the board to your Computer or Raspberry PI through microUSB port(Note: The USB of the board cannot supply power to the MCU, so an external 12/24V is required to supply power to the board).
2. Press and hold the `Boot` button on the back, then click the `Reset` button, and then release the `boot` button. And The MCU has entered DFU mode now.
   <br/><img src=Images/boot.png width="400" /><br/>
3. Update the firmware through Computer
   * Download `firmware.bin` into MCU with `STM32CubeProgrammer`, and then click `Reset` button to enter normal working mode.
4. Update the firmware Directly through Raspberry PI
   * from your ssh session, run `lsusb`. and find the ID of the dfu device.
   * run `make flash FLASH_DEVICE=1234:5678` replace 1234:5678 with the ID from the previous step
   * click `Reset` button to enter normal working mode
## Configure the printer parameters
1. [HermitCrab_Canbus_pins.cfg](./HermitCrab_Canbus_pins.cfg) is the configuration file of klipper which contains all pinouts of Canbus hotend board
2. If you use USB to communicate with raspberry pi, run the `ls /dev/serial/by-id/*` command in raspberry pi to get the correct ID number of the motherboard, and set the correct ID number in `printer.cfg`. And wiring reference [here](#raspberry-pi-communicates-with-the-hotend-board-via-usb)
    ```
    [mcu HermitCrab]
    serial: /dev/serial/by-id/usb-Klipper_stm32f072xx_1234
    ```
3. If you use Canbus to communicate with raspberry pi, you need to modify the following files by SSH command. And wiring reference [here](#raspberry-pi-communicates-with-the-hotend-board-via-canbus)<br/>
   Refer to [klipper's official Canbus config](https://www.klipper3d.org/CANBUS.html) to set Canbus
   * Input `sudo nano /boot/config.txt` command, input the password (default: raspberry), Then add the following content to the `config.txt` file
     ```
     dtparam=spi=on
     dtoverlay=mcp2515-can0,oscillator=12000000,interrupt=25,spimaxfrequency=1000000
     ```
     Save(`Ctrl + S`) and Exit(`Ctrl + X`) after modification, input `sudo reboot` to restart raspberry pi
   * Input the `dmesg | grep -i '\(can\|spi\)'` command to test whether MCP2515 has been connected normally after the restart is completed.<br/>
     The normal response should be as follows
     ```
     [ 8.680446] CAN device driver interface
     [ 8.697558] mcp251x spi0.0 can0: MCP2515 successfully initialized.
     [ 9.482332] IPv6: ADDRCONF(NETDEV_CHANGE): can0: link becomes ready
     ```
     <img src=Images/can_connected.png width="800" /><br/>
   * Input `sudo nano /etc/network/interfaces.d/can0` command to create a new file named `can0`, and write the following content
     ```
     auto can0
     iface can0 can static
         bitrate 250000
         up ifconfig $IFACE txqueuelen 1024
     ```
     Set the Canbus speed to 250K (consistent with the speed set in [firmware-F072-Canbus.bin](./firmware-F072-Canbus.bin)), and set the `txqueuelen` to 1024 bytes. Save(`Ctrl + S`) and Exit(`Ctrl + X`) after modification, input `sudo reboot`to restart raspberry pi

   * Each micro-controller on the CAN bus is assigned a unique id based on the factory chip identifier encoded into each micro-controller. To find each micro-controller device id, make sure the hardware is powered and wired correctly, and then run:<br/>
     `~/klippy-env/bin/python ~/klipper/scripts/canbus_query.py can0`<br/>
     If uninitialized CAN devices are detected the above command will report lines like the following:<br/>
     `Found canbus_uuid=0e0d81e4210c`<br/>
     set the correct ID number in `printer.cfg`<br/>
     ```
     [mcu HermitCrab]
     canbus_uuid: 0e0d81e4210c
     ```
