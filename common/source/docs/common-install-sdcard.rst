.. _common-install-sdcard:

[copywiki destination="copter,plane,rover,planner,blimp,sub,dev"]

============================
Loading Firmware via SD Card
============================

It is possible to update the ArduPilot firmware on certain autopilots by placing a specifically-named file onto an SD card and running the autopilot's bootloader (e.g. by power-cycling the board).

.. note:: at this time only a few autopilots have this capability. See instructions below to determine if an autopilot has this feature. Only H7 based autopilots using MMC connected SD cards are capable of this and not all that do have had the feature added as of yet.

Why would I want to do this?
============================
There are several reasons this technique may be of use:

  - There is no convenient, reliable access to the vehicle's USB port
  - It is desired to update the vehicle firmware over a telemetry radio
  - The vehicle is remote and the local operators may struggle with the normal interfaces to update the firmware. This allows OEMs to remotely add an updated file onto users' SD cards which would upload on the next boot of the vehicle.

Is my autopilot capable of this?
================================
In order to do this several elements must be present:

  - The autopilot must have SD card capability
  - The SD card must be enabled in its bootloader
  - The bootloader code and flashing firmware must be enabled in the autopilots firmware definition files (hwdefs)

To see if a given autopilot has this capability, you can check in its firmware folder on the `Firmware build server <https://firmware.ardupilot.org>`__ to see if it has a ``xxxxxx.abin`` file present.

The Bootloader
==============
A bootloader capable of flashing from the SD card is required (and the autopilot must have SD card capability, obviously). Newer autopilot's with SD cards may or may not have this capability and a capable bootloader included in their hardware definition (if not, see :ref:`Building capable firmware yourself <building-autoflash-firmware>`).

If the autopilot firmware currently provides this capability, updating your current bootloader may be required to support flashing from SD card.  Note that updating the autopilot's bootloader is an operation which can make your board non-operational, and difficult to recover.  More-so with boards that do not expose a "boot0" pin, such as the CubeOrange.  Be aware of this risk, and be prepared to spend considerable time recovering a board if something bad happens when updating the bootloader.

:ref:`The instructions on updating the bootloader <common-bootloader-update>` can be followed to update your bootloader; be aware that you must use a recent firmware (4.5 or higher) to obtain a suitable bootloader.

The Firmware File
=================
The file to be loaded onto the SD card is of a specially named "xxxxx.abin" file created for this purpose.  This is a binary file with a small amount of text providing some information about the binary. Most notably, a checksum which the bootloader will verify before attempting to flash the board.

Boards which support flash-from-sdcard will also have ``.abin`` files available for download from the `Firmware build server <https://firmware.ardupilot.org>`__.

Firmware File Name
==================
The filename which is used when generating firmware is ***not*** the correct name to use when placing the firmware on the SD card.  When the ``.abin`` files are generated they contain the vehicle name, for example, "arduplane.abin".

There is only one correct filename that may be used to flash-from-sdcard; this is ``ardupilot.abin``.  When placing the file on the SD card, ensure the file has been renamed to ``ardupilot.abin``.

Transferring the File to the SD card
====================================
This can be done in your operating system as you would ordinarily interact with the SD card (e.g. file browser).

You can also transfer the file to the SD card's base directory via ``MAVFTP`` using Mission Planner or similar GCS

.. image:: ../../../images/MP-install-firmware-sdcard.png
    :target: ../_images/MP-install-firmware-sdcard.png

Ensure the file is the correct size before continuing.

After the transfer is complete, the directory listing should look something like this:

  ::

        RTL> ftp put /home/pbarker/arducopter.abin ardupilot.abin
        RTL> Putting /home/pbarker/arducopter.abin as ardupilot.abin
        Sent file of length  1847687
        RTL> ftp list
        RTL> Listing /
         D APM
           ardupilot.abin	1847687
        Total size 1804.38 kByte

Triggering the Flash Update
===========================
Power cycle the board to enter the bootloader which will automatically check for the firmware update file and begin flashing it.  Alternatively to avoid the powercycle the autopilot may be rebooted via a `PREFLIGHT_REBOOT_SHUTDOWN <https://mavlink.io/en/messages/common.html#MAV_CMD_PREFLIGHT_REBOOT_SHUTDOWN>`__ command with the 'Param1' field set to 3 (ie. "Reboot autopilot and keep it in the bootloader until upgraded")

It should take roughly 1 minute to verify the firmware and flash it to the vehicle's internal flash.

If the process completes successfully the file will renamed to ``ardupilot-flashed.abin``.  The vehicle should proceed to boot the firmware once flashing is complete.

Troubleshooting
===============
Several things can go wrong with the firmware flash, but some diagnostics are available to help work out what the problem might be.

  - At each stage of the flashing process, the ``ardupilot.abin`` file is renamed to reflect the stage
  - if the file is called ``ardupilot-verify.abin`` then the process failed when trying to checksum the file, or the board was interrupted when doing so.
  - if the file is called ``ardupilot-verify-failed.abin`` then the checksum the bootloader calculated did not match the bootloader in the ``.abin`` metadata.
  - if the file is called ``ardupilot-flash.abin`` the process failed when writing the firmware, or the board was interrupted while doing so.  The board is unlikely to boot into an ArduPilot firmware if this has happened, so a re-flash will be required.
  - if the file is called ``ardupilot-flashed.abin`` you should not need this "troubleshooting" section, as the flash process has succeeded!

.. _building-autoflash-firmware:

Building the firmware yourself
==============================

If the autopilot has an SD card capability but no SD Card flash-able firmware is present on the `Firmware build server <https://firmware.ardupilot.org>`__, you can build the firmware yourself. You must setup a build environment and then modify the autopilot's hwdefs to build a capable bootloader and an ``xxxx.abin`` firmware, see :ref:`building-the-code`.

In the ``hwdef-bl.dat`` file you must include this:

.. code::

    define AP_BOOTLOADER_FLASH_FROM_SD_ENABLED 1
    define FATFS_HAL_DEVICE SDCD1
    define HAL_OS_FATFS_IO 1
    # FATFS support:
    define CH_CFG_USE_MEMCORE 1
    define CH_CFG_USE_HEAP 1
    define CH_CFG_USE_SEMAPHORES 0
    define CH_CFG_USE_MUTEXES 1
    define CH_CFG_USE_DYNAMIC 1
    define CH_CFG_USE_WAITEXIT 1
    define CH_CFG_USE_REGISTRY 1


* Also you must include the SD card setup. See the `MatekH743 autopilot definition <https://github.com/ArduPilot/ardupilot/blob/master/libraries/AP_HAL_ChibiOS/hwdef/MatekH743/hwdef-bl.dat>`__ for an example.

In the ``hwdef.dat`` file you must include this:

.. code::

    env BUILD_ABIN True

When you build the firmware you will see, note the .abin file is created:

  ::

         BUILD SUMMARY
    Build directory: /home/pbarker/rc/ardupilot/build/CubeOrange
    Target         Text (B)  Data (B)  BSS (B)  Total Flash Used (B)  Free Flash (B)  External Flash Used (B)
    ---------------------------------------------------------------------------------------------------------
    bin/arduplane   1868612      3536   258740               1872148           93928  Not Applicable

    Build commands will be stored in build/CubeOrange/compile_commands.json
    'plane' finished successfully (24.283s)
    pbarker@fx:~/rc/ardupilot(master)$ ls -l build/CubeOrange/bin
    total 18792
    -rwxrwxr-x 1 pbarker pbarker 3135448 Sep 29 19:15 arduplane
    -rw-rw-r-- 1 pbarker pbarker 1872247 Sep 29 19:15 arduplane.abin
    -rw-rw-r-- 1 pbarker pbarker 1684192 Sep 29 19:15 arduplane.apj
    -rwxrwxr-x 1 pbarker pbarker 1872152 Sep 29 19:15 arduplane.bin
    -rw-rw-r-- 1 pbarker pbarker 5148900 Sep 29 19:15 arduplane.hex
    -rw-rw-r-- 1 pbarker pbarker 5509380 Sep 29 19:15 arduplane_with_bl.hex

Demo Video
==========

.. youtube:: hCdXe1UTjK4
  :width: 100%
