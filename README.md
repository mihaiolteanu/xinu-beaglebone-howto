[Xinu](http://www.xinu.cs.purdue.edu/) is the elegant, minimalistic operating system.

# Download xinu.boot on the BeagleBone

There are already [instructions](http://www.xinu.cs.purdue.edu/files/Xinu_BBB_instructions.txt) on the Xinu official page on how to download and boot Xinu from the BeagleBone Black(BBB). But I've had the following issues with it:
- minicom wasn't able to download the file
- after booting into Xinu, the watchdog would reset the BBB after ~60s

So here's what I did.

First, the instructions are only valid with the serial communication program **minicom**. There are other ways to download the boot image, but I'm only concerned with this one.

`$ sudo apt-get install minicom`

The [serial cable](http://dave.cheney.net/2013/09/22/two-point-five-ways-to-access-the-serial-console-on-your-beaglebone-black) should be connected to the BBB. This will (probably) show as **/dev/ttyUSB0**. Make sure you see it.

Enter the minicom setup with:

`$ minicom -s`

            +-----[configuration]------+
            | Filenames and paths      |
            | File transfer protocols  |
            | Serial port setup        |
            | Modem and dialing        |
            | Screen and keyboard      |
            | Save setup as dfl        |
            | Save setup as..          |
            | Exit                     |
            | Exit from Minicom        |
            +--------------------------+

Select *Serial port setup* 

    +-----------------------------------------------------------------------+   
    | A -    Serial Device      : /dev/ttyUSB0                              |   
    | B - Lockfile Location     : /var/lock                                 |   
    | C -   Callin Program      :                                           |   
    | D -  Callout Program      :                                           |   
    | E -    Bps/Par/Bits       : 115200 8N1                                |   
    | F - Hardware Flow Control : No                                        |   
    | G - Software Flow Control : No                                        |   
    |                                                                       |   
    |    Change which setting?                                              |   
    +-----------------------------------------------------------------------+   

Press Shift+a (or A, for short) to change the *Serial Device* to /dev/ttyUSB0.
Press F to disable the *Hardware Flow Control*. (it will automatically toggle to *No*).
Press Esc to go to the previous menu and then *Save setup as dfl*. This will save the previous settings for the next time minicom is launched.

Lastly, the ymodem protocol used by minicom for file transfer uses /usr/bin/rb and /usr/bin/sb by default. If you don't have those installed, you'll get the `failure executing protocol` error when trying to upload the boot file. The easy fix is (thanks [Andrew](https://axixmiqui.wordpress.com/2008/05/16/minicom-ymodem-issue/)!):

`$ sudo apt-get install lrzsz`

Run minicom with noinit as admin:

`$ sudo minicom -o`

Next, follow steps 4-8 from the [instructions](http://www.xinu.cs.purdue.edu/files/Xinu_BBB_instructions.txt) mentioned above. If you somehow can't interrupt the boot process because you can't send any keys through minicom, check the "serial port setup" again.

Sometimes you'll get a `timeout on pathname` error from minicom. Just try again.

Do not follow step 9 yet. That is, do not issue the `bootm` command. If the watchdog is enabled on the BBB, after the Xinu boots, the watchdog timer will expire and the BBB will reset. Xinu does not have a "feed-the-dog" implementation as far as I can tell. The watchdog timeout is approximately 60 seconds.

# Disable the BBB watchdog from uboot

You can check if the watchdog is set by reading the **WDT_WSPR** register, while in uboot (see also the [Technical Reference Manual for AM335x](http://www.ti.com/general/docs/lit/getliterature.tsp?baseLiteratureNumber=spruh73&fileType=pdf)).
The WDT_WSPR has an offset of 48h from the base register, WDT1, which has the address `0x44e35000`. So we need to check the 0x44e35048. 
From uboot (don't use the exact address, only 10h increments, otherwise you might get a reset):

`U-Boot# md 0x44e35030 8`

>44e35030: 00001234 00000000 00000000 00000000    4...............
>
>44e35040: 00000000 00000000 00004444 00000000    ........DD......

In this case, the value is 0x4444, which means the watchdog is enabled.
To disable it,

`U-Boot# mw 0x44e35048 0xaaaa; sleep 1; mw 0x44e35048 0x5555`

There is only one trick though. The register WDT_WWPS.W_PEND_WSPR (offset 34h) must have a value different than zero when the second write (0x5555) is issued. Repeat the two mw steps above if the watchdog was not disabled.

Verify that the watchdog was disabled (i.e. the register was succesfully writen to with the value 0x5555)

`U-Boot# md 0x44e35030 8`

>44e35030: 00001234 00000000 00000000 00000000    4...............
>
>44e35040: 00000000 00000000 00005555 00000000    ........UUUU....

Now run the last step,

`U-Boot# bootm`
