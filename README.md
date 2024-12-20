# Tested using Debian 12

## Disclaimer
This repo is a fork of 4ban's repo. His guide sadly didn't work for me, firstly because I use linux and not Mac, and secondly there is probably a difference in hardware revisions or firmware versions on these, as I had to use AP55C instead of AP100C.  
I take no credit, 4ban did all the hard work, I just customized it to work for me, and hopefully others.  
You can find the original repo here: [GitHub 4ban](https://github.com/4ban/Sophos_AP_55C_OpenWRT)

## What you need
* ethernet cable
* screw driver
* USB to TTL serial UART (I used FT232)
  
## Step 1. Prepare your Sophos.
Disassemble it and find the UART header. 
![IMG_3812](https://github.com/user-attachments/assets/23fda168-6636-4b02-bca3-191f4a10e13e)

Header pins on the picture from left to right: 
* pin 1 - RX
* pin 2 - TX
* pin 3 - GND
* pin 4 - 3.3v (won't be used)

Your USB-to-TTL adapter should be connected to the header. GND to GND, RX to TX, TX to RX, leave 3.3V unconnected.

1. Connect your USB-to-TTL adapter to the AP and connect it to the PC usb port. At this point we don't powerup the AP yet.

2. Command `ls /dev/tty.*` supposed to show the device now: `/dev/ttyUSB0` (or similar)

5. Now you can open a separate terminal window and run screen: `sudo screen /dev/ttyUSB0 115200` 

## Step 2. Configure Ethernet
Connect the Sophos AP 55C to your PC via Ethernet cable. Assign a Static IP to your PC's Ethernet adapter:

* IP Address: `192.168.99.8`
* Subnet Mask: `255.255.255.0`
* Router: Leave blank.

## Step 3. Time to power it up
UART and Ethernet are prepared. Time to power on the AP. 

Start hammering `4` as soon as you power up the AP. You'll see the console `ath>` after a little while.

## Step 4. Download firmwares (latest I used 23.05.5)
[sophos_ap55c-initramfs-kernel.bin](https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-23.05.5-ath79-generic-sophos_ap55c-initramfs-kernel.bin)
[sophos_ap55c-squashfs-sysupgrade.bin](https://downloads.openwrt.org/releases/23.05.5/targets/ath79/generic/openwrt-23.05.5-ath79-generic-sophos_ap55c-squashfs-sysupgrade.bin)

The files are also available in this repo.

## Step 5. Set Up a TFTP Server
* Install TFTP server and client: `sudo apt install tftpd-hpa tftp-hpa` 
* The TFTP server will by default serve files from: `/srv/tftp`
* Copy the files to the tftp folder: `cp ~/Donwloads/openwrt-23.05.5-ath79-generic-sophos_ap55c-initramfs-kernel.bin /srv/tftp`
* Rename the initramfs-kernel file to `uImage_AP55C`, or just use the already renamed file from this repo.
* Test the TFTP server with these commands (make sure you are not in the /srv/tftp folder, as this will erase the original file):
```
tftp localhost
tftp> get uImage_AP100C
tftp> quit
```

## Step 6. Load firmware to the AP
If all the above was configured correctly. 

We can try to upload the firmware to the AP. The command is running on the AP where we have `ath>` prompt:

* `setenv serverip 192.168.99.8`
* `setenv ipaddr 192.168.99.9`
* `setenv bootfile uImage_AP55C` (for some reason the default is uImage_AP100C)
* `printenv` - to check we saved the env for tftp server address
* `tftpboot` - this one supposed to load the firmware from our tftp server.

You should see progress logs showing the firmware is being loaded into RAM at address `0x81000000`

We need to set the firmware address:
* `setenv fwaddr 0x9f070000`
* `erase $fwaddr +$filesize` - Here we erase the current firmware from the flash (Dangerous step, not sure we can revert this somehow)
* `cp.b $fileaddr $fwaddr $filesize` - and write the uploaded firmware to flash
* `iminfo $fwaddr` - everything should be ok 

## Step 7. Boot
run `boot` command on the AP and it supposed to reboot and boot to the openwrt (you'll see logs).

## Step 8. Change Ethernet again
Set your PC’s Ethernet back to DHCP auto.

Disconnect the UART pins from the AP and USB from the PC, and reboot the AP.

## Step 9. Install sysupgrade firmware
Open a browser and go to http://192.168.1.1 (there is no password by default)
>user: root  
>password: [no password]

When you login you will be greeted with two warnings:
1. No password set
2. Firmware running in initramfs mode  

Install new firmware:
1. Click on the firmware button to go to the Flash operations page.
2. At the bottom under `Flash new firmware image` click on `Flash image`
3. Click `Browse` and select the `squashfs-sysupgrade.bin` file
4. Click `Upload` and wait for it to complete
5. Leave everything as it is and click `Conmtinue`. WARNING: This will actually start flashing.

## Thats it!
Now you can start to configure everything to your needs, but you should definately start by setting a password.