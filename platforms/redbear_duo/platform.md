Redbear Duo SDU platform
==========
This secure device update platform supports the [RedBear Duo](https://github.com/redbear/Duo) target device, adding support for secure Bluetooth BLE Over The Air firmware udpates for your Duo Arduino sketch.

The RedBear Duo SDU platform uses the secure firmware download protocol used by Nordic Semiconductor in their open source [Android-nRF-Toolbox](https://github.com/Trellis-Logic/Android-nRF-Toolbox) application and associated [Android](https://github.com/Trellis-Logic/Android-DFU-Library) and [iPhone](https://github.com/Trellis-Logic/IOS-Pods-DFU-Library) libraries.  This allows you to easily either use existing applications for Android and iPhone to provide firmware updates for your RedBear Duo or add this functionality to your own Android or iPhone applications.

This project is not sponsored or endorsed by RedBear or Nordic Semiconductor.  All RedBear related trademark names including Duo are owned by RedBear.  All Nordic related trademark names including nrfutil and nRF Toolkit are owned by Nordic Semiconductor

### SDU Library Demo

The demo instructions below demonstrate the capabilities of the SDU software with tools used to sign firmware update packages and download to your Duo over bluetooth.  The instructions use a predefined [test_key](demo/test_key) private key, two SDU sample signed firmware binaries [BLE_BlinkFast.zip](demo/BLE_BlinkFast.zip) and [BLE_BlinkSlow.zip](demo/BLE_BlinkSlow.zip) signed with this key, and a binary payload in [BLE_Blink_Fast.bin](demo/BLE_Blink_Fast.bin).  Each of the BLE files are built based on the [SimpleBLEPeripheral](https://github.com/Trellis-Logic/STM32-Arduino/tree/master/arduino/libraries/RedBear_Duo/examples/03.BLE/SimpleBLEPeripheral) RedBear Duo example code modified with integration of the SDU library.

The instructions assume you've configured your RedBear Duo for use with Arduino based on the instructions in the [RedBear Duo Getting Started with Arduino IDE](https://github.com/redbear/Duo/blob/master/docs/getting_started_with_arduino_ide.md) guide.

You should also install the [dfu-util](https://github.com/redbear/Duo/blob/master/docs/dfu-util_installation_guide.md) utility from Redbear for device firmware update.


#### Download the Initial SDU Example bin file to the DUO

In this section, we'll setup your Duo with the initial file it needs to be able to support Bluetooth OTA using the nRF Toolbox application

1. Put your duo in DFU mode
 * Hold down buth buttons
 * Release only the RESET button, while holding down the SETUP button.
 * Wait for the LED to start blinking yellow
 * Release the SETUP button

2. Using dfu-util, download the .bin file to the device, using command:  
```
dfu-util -d 2b04:d058 -a 0 -s 0x80C0000 -D demo/BLE_Blink_Fast.bin
```  
where demo/BLE_Blink_Fast.bin can be downloaded [here](demo/BLE_Blink_Fast.bin)  
3. Reset the duo.  You should now see the blue LED flashing about 10 times per second.

#### Upload SDU Firmware via the nRF Toolbox

1. Install nRF-Toolbox for android or iphone from [The AppStore or Google Play](https://www.nordicsemi.com/eng/Products/Nordic-mobile-Apps/nRF-Toolbox-App)
2. Download the two signed firmware binaries [BLE_BlinkFast.zip](demo/BLE_BlinkFast.zip) and [BLE_BlinkSlow.zip](demo/BLE_BlinkSlow.zip), as well as the binary payload in [BLE_Blink_Fast.bin](demo/BLE_Blink_Fast.bin) to your phone.  You may want to email them to yourself.
3. Open the nRF-Toolbox and select the "DFU" button
4. Select the payload zip file [BLE_BlinkSlow.zip](demo/BLE_BlinkSlow.zip)
5. Select the Duo.  It should be listed as "RBL-DUO", may also be displayed as "BLE Peripheral" on the iPhone application.
6. Click upload.  The payload should transfer to the device successfully.  After a few seconds, the LED should start to blink more slowly (approximately once per second) indicating a new firmware payload is now in use.
7. If desired, repeat the steps to switch between the [BLE_BlinkFast.zip](demo/BLE_BlinkFast.zip) and [BLE_BlinkSlow.zip](demo/BLE_BlinkSlow.zip) payloads to confirm device update over Bluetooth is working as expected.

#### Sign Your Own Firmware Image

In this section, we'll sign your own firmware image with the [test_key](demo/test_key) used by the SDU example firmware.  This will allow your firmware image to be transferred successfully over bluetooth and programmed on the RedBear Duo.  Note that since we haven't yet integrated the SDU library with your application, you will need to perform future updates using the dfu-util or Arduino environment.  This first test just verifies your download file can be transferred and programmed successfully using the sample firmware application.

1. Start by cloning the [nrfutil](https://github.com/Trellis-Logic/pc-nrfutil) application and using the instructions to install on your system.
2. Export your sketch in Arduino using Sketch->Export Compiled Binary
3. Sign your sketch binary with the test key using:  
```
nrfutil pkg generate --application <path_to_bin_file> --key-file demo/test_key myapplication.zip
```
4. Use the instructions above with the nRF Toolbox application to download your firmware image to the RedBear Duo.  You should now see your application running on the Duo.

#### Incorporating the SDU Library Into Your Application
In this section we'll include the SDU library and a custom key into your Arduino application so you can provide updates to your own application over Bluetooth.  

##### Add the library to your sketch
1. Open your Arduino Sketch.
2. Select Sketch->Include Library->Add Zip Library and add the sdu.zip file.
Note: the SDU library build for RedBear Duo Arduino is currently in limited beta, please contact me at danwalkes@trellis-logic.com if you'd like to be a part of the initial test group.  
3. Select Sketch->Include Library->sdu.

##### Create your own signing key
1. Create a signing key specific to your application using the [nrfutil](https://github.com/Trellis-Logic/pc-nrfutil) application and options:  
```
nrfutil keys generate myprivatekey.pem
nrfutil keys display --key pk --format code myprivatekey.pem
```  
2. Save the private key myprivatekey.pem in a secure and backed up location.  *Anyone with this key will be able to update your device firmware!!*  *If you loose this key you will not be able to build new firmware for your device!!*
3. Copy the resulting output with declaration of crypto_key_pk value to your sketch file and use with the integration steps in the next step of the process.  

##### Integrate the SDU Library With Your Application
See example integration in [this commit](https://github.com/Trellis-Logic/STM32-Arduino/commit/99097785a01446489b8b681e810621610a9af758).  Integration consists of a few simple steps:  
1. Adding an advertisement for the BLE_SDU_SERVICE_UUID.  
2. Allocating memory for the sdu_context structure, for instance as a global varible.  
3. Add an initialization call to sdu_ble_redbear_transport_init in the setup() function, after adding any other characteristics but before setting up advertising parameters.  Pass in a pointer to the private key structure you've created in the previous step.  
4. Add a call to sdu_update in the loop()function, passing the sdu_context structure.  
5. Add a call to the sdu_gatt_write_callback in your gatWriteCallback handler.  
6. Build and test your project.  If you see an error message "dynalib location not correct" please use the notes in the troubleshooting section below to resolve.  
7. Update your project over USB the first time, since your signing key has changed from the test key.  
8. Sign firmware binary files with your new key using:  
```
nrfutil pkg generate --application <path_to_bin_file> --key-file myprivatekey.pem myapplication.zip
```  
and upload to the device using nRF Toolbox or the iPhone/Android Libraries with your own mobile application.

### Troubleshooting
1. Failures during download with the Android application  
  * The Android DFU library does not correctly retry CRC failures before [this pull request](https://github.com/NordicSemiconductor/Android-DFU-Library/pull/41).  Rebuild with the latest source for the DFU library to resolve.
2. Message "dynalib location not correct" when attempting to build your project with the sdu library.  
  * See instructions [on the RedBear forums](http://discuss.redbear.cc/t/dynalib-location-not-correct-linker-error-on-arduino-build/1639) to patch your linker command file.  
3. Red blinking hardware fault LED pattern after building the project with RedBear SDK version 0.3.0 or 0.3.1
 * See [this issue](https://github.com/redbear/STM32-Arduino/issues/21) and [this fix](https://github.com/Trellis-Logic/STM32-Arduino/commit/78d96cfcea16f28fd67bb8d5520bad11679a8ab6) to the C:\Users\Username\AppData\Local\Arduino15\packages\RedBear\hardware\STM32F2\0.3.X\platform.txt file.
