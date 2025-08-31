# Table of Contents

- [Intro - APC-SmartUps-Firmware-Recovery](#Intro)
- [How did I get here?](#How)
- [Fixing it](#Fix)
- [Tools required for recovery](#Tools)
- [Getting EEPROM Data](#Get-EEPROM)
- [Flashing EEPROM Data](#Put-EEPROM)
- [It's still broken - Next Steps](#Not-Fixed)
- [Recovering via Serial](#Serial-Recovery)
- [Additional Info](#Additional)
- [Links](#Links)

#

<a name="Intro"/></a>
# Intro - APC-SmartUps-Firmware-Recovery
Repo with steps and process for recovering a corrupted UPS - These are the steps I took with an SMT1500 ID17. (Using a Donor SMT1000 ID17's firmware)
These steps may or may not work for you. Do this at your own risk (Assuming you've got a bricked UPS, the risk is already low I suppose?)

<a name="How"/></a>
## How did I get here?
I decided to update my UPS firmware. My last firmware update in the workplace went so well, i thought, may as well update my personal UPS'.
I have 2 to update. An APC ID18 SMT1000 and an APC ID17 SMT1000. The update on the ID18 went okay. The update on ID17. I went away from my laptop that was doing the updrade... it went into standby mode mid update, and the rest... well it's a bricked UPS.

<a name="Fix"/></a>
# Fixing it

I'd recommend trying [Recovering via Serial](#Serial-Recovery) first. as using a cable to flash your ups is way easier than a teardown. Especially if it's not required.

<a name="Tools"/></a>
## Tools required for recovery
- RS-232 to RJ-50 940-0625A (If EEPROM flash didn't resolve)
  - USB - RS-232 Serial Adapter (Needed to use above cable)
- A Computer
- ST-LINK V2 (And appropriate software)

<a name="Get-EEPROM"/></a>
## Getting EEPROM Data

The Following steps are if you have a donor UPS that will be providing a copy of it's firmware.

You will need to tear down the UPS and get access to the mainboard.
On the Mainboard there is a 10 pin header labelled J602.

It's pinout seen here:

![JTAG 10 pin header pinout](../main/Pictures/JTAG%20pinout.png?raw=true)

1. Connect the ST-LINK to the UPS.
   - There is usually a number 1 for pin 1 (Which will be VCC).
   - If you're unsure, you can double check by finding the ground pin (Labelled GNDDetect in the image) with a multimeter in continuity mode. With one probe on a screw hole and the other probing for a non-resistive continuity.
   - The buzzer on the UPS may sound when you connect the STLINK. It may also change tone as you connect/disconnect and read.
   - If you have any issues, double check your wiring. Most issues will be down to this. The ST-LINK will be able to talk to the chip and see which one it is as well as the data on it. (I think mine was an STM32F103RC Medium Density with 128kb, But the ID18 is a High Density with 256kb of storage)
2. Pull EEPROM Data
   - Once connected and open the software to pull the EEPROM data.
   - You will need to set the Address to 0x00000000. If you set it to 0x08000000 then when you write to file. the filesize will be 169kb instead of the limited EEPROM size of 128kb (Specifically for the 4.5g/ID17. The ID18 seems to have 256kb. But this issue may still cause data size issues)

  ![ST-Link Software Screenshot](../main/Pictures/st-link.PNG?raw=true)
    
3. Save EEPROM Data
   - Once done. save the output as a .bin file. (Can be .hex, but it's difficult to guage if filesize will be actual size of data written and may have other issues doing it this way)
   - I've saved mine and uploaded it to this repo here: [Repo](../main/RAW%20Binary%20FW) - [File](../main/RAW%20Binary%20FW/ID17%20FW5.0.bin)
   - There are other RAW FW files available in that folder. Mostly for ID18 SMT UPS'.

<a name="Put-EEPROM"/></a>
## Flashing EEPROM Data
Flashing the EEPROM is the same as getting the EEPROM data. Except you open a file in the programmer program and you write the new firmware on. (The software will usually manage erasing, writing and verifying.)
There was nothing special nor extra you need to know. If you get a file size warning. Make sure the file size and chip size match and that you have the correct firmware.

<a name="Not-Fixed"/></a>
## It's still broken - Next Steps
Mine was still broken, yours may be too. It may be due to mismatch of firmwares as these things have multiple firmware's. I.e. MCU, UBL, MBL & UPS. or something else.
Mine would power on but the screen wouldn't do anything and the lights didn't do anything.
Only one of the steps below worked for me, but you could try them. Which I'll provide further documentation for.

Steps to try:
- USB
  - Plug in a USB, see if your PC detects it. If it does, flash it using APC's Firmware Updater Tool
- NMC
  - See if your NMC can see the UPS. As this communicates to the UPS via serial, or various other methods. If it can, reflash using NMC
- Serial
  - Plug in your serial adapter and the RJ50 cable into your UPS (Mine says UPS Monitoring Port) and see if anything appears on the terminal. If it does. There is hope. Go to the next section.

<a name="Serial-Recovery"/></a>
## Recovering via Serial
This may take a few goes.

On my serial terminal I saw this:

![Serial Capture](../main/Pictures/SerialSS7.2-No-App-in-Mem.png?raw=true)

(Originally the version was showing 5.0)
The Terminal should show progressive C's appearing waiting for presumably an xmodem transfer. (Shoutout to Schneider forum members for this info)
Use a terminal like ExtraPutty or Tera Term (for windows) or Minicom for linux

For visual queues, the following occurs.
AC Green light flashes when communicating via xmodem.
Battery AC Orange and Battery Fault Red Light are Solid/On while something is being programmed.
General Red Warning Light flashes indicating flashing/programming.

If all lights are indicating, generally a good sign.
If green light only is indicating, may mean something's gone wrong. Keep reading for further info.

See an example video [here](../main/Videos/VID-20250831-WA0001.mp4?raw=true)

The first sequence is it programming. the second sequence is it just receiving and ignoring.

To flash the UPS, We use XModem to send the .enc file from the APC website. In my case I was using [SMT17UPS_07-1.enc](../main/APC%20provided%20FW%20and%20Release%20Notes/SMT17UPS_07-1.enc)

Process I went through:
1. Initial transfer started
   - The UPS restarted on me after 300 odd packets out of 1347.
   - The XModem transfer failed
2. Second transfer started
   - The video above occured.
   - Transfer never ended (Due to lost ACK response, or lost sync)

![Over-sent XModem packets](../main/Pictures/Over-Send.png?raw=true)

3. Third Transfer Started
   - The UPS flashed some data and stopped around 500 packets
   - XModem transfer didn't stop. thought it was same scenario as before
   - XModem transfer finishes. I see "UPS Application" text appear in terminal before it stops responding at all.
  
4. Power on UPS. It's up and running.

As long as the serial port is active and responding then you can send the firmware as many times as you want.

<a name="Additional"/></a>
## Additional Info

- Some elements of information will be re-iterated here
- As long as the serial port is active and responding then you can send the firmware as many times as you want.
- The serial port seems to be inactive during normal operation of the UPS.
- Apparently according to some posts online, The UPS will receive an XModem transfer even if there is no indication of the serial port being active.
- Apparently, the serial port is active and will take XModem transfers upon startup.
- You may not need the ST-LINK to unbrick your UPS. The XModem transfer may be enough. Unfortunately, I couldn't recover that way.

Sources, and Links to recovering UPS'. Also available in text document called [Sites that helped.txt](../main/Sites%20that%20helped.txt) Found at the root of this repo.

<a name="Links"/></a>
## Links

https://www.tarball.ca/posts/unbricking-an-apc-smt3000-ups-after-a-failed-firmware-update

https://electronics.stackexchange.com/questions/581274/why-is-there-a-difference-between-arm-10-pin-jtag-debug-pinout-and-the-stlink

https://secure.ups-trader.co.uk/Forum/topic/38-how-to-un-brick-a-smt-range-ups-after-firmware-flash-failure.html

https://www.eevblog.com/forum/repair/help-with-bricked-apc-ups-smt1500/50

https://wejn.org/2021/02/howto-usb-cable-for-APC-Smart-UPS-SC450RMI1U

https://community.se.com/t5/APC-UPS-Data-Center-Enterprise/APC-UPS-SMX1500-firmware-update-failure-and-fixed/td-p/294993


