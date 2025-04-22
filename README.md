# Klipperized Input Shaping for the MK3S/+, a simplified guide for a powerful upgrade

Latest  updates:

4/2025
- Guidance update
  - Reworked PrusaSlicer settings based on latest version (2.9.2 at time of writing).
  - Updated IS instructions based on current Klipper.
- Macros update
  - Print start and purge chagned to be more like the MK4.
  - Print end will raise toolhead to at least 50mm for easier access to clean leftover filament before next start.
  - Filament runout reuses unload filament macro and parks in a more favorable location.
  - Filament unload uses MK3.5 sequence for cleaner break
- IS update - Include support for BIGTREETECH (BTT) ADXL345 accelerometer board.
- Enable adaptive bed mesh calibration
- Stepper settings for current and microsteps set to MK3.5 values

4/17/2024 - Guidance update - In PrusaSlicer, Print Settings > Advanced: DISABLE "Arc Fitting" - G2/G3 should not be used with Klipper and is not necessary.

4/2/2024 - Removed Mechanical Gantry Calibration macro. I'm not sure if it ever worked properly.

4/1/2024 - Fixed missing tabs on PRUSA_LINE_PURGE in macros, and removed duplicate gcode_arc lines in einsy-rambo.

3/27/2024 - Filament runout code updated to prevent double pause

3/26/2024 - Removed duplicative Pause/Cancel/Resume macros from macros.cfg

3/25/2024 - Added "pullups" to einsy-rambo.cfg


Tired of seeing everyone upgrading to MK4s and feeling like you're missing out on the action? This project is an attempt to create simple, easy-to-follow documentation for performing an inexpensive, simple, and powerful upgrade to your Prusa MK3S/+. Originally based on work by dz0ny (https://github.com/dz0ny/klipper-prusa-mk3s).

WARNING: This upgrade involves the use and installation of Klipper, which is a significantly different experience than Prusa's firmware. Think Klipper = Custom Android, vs. Prusa = Apple iOS. Klipper is powerful, but can be potentially dangerous if misconfigured. We take no responsibility for damage or injury. This guide and configuration files may contain errors or omissions. 

**As of 3/31/2024 - Current discussion and live help can be found here, may need to join Klipper Discord first:** https://discord.klipper3d.org/ in this channel: https://discord.com/channels/431557959978450984/1205590905378312293

**There are several reasons you should try this upgrade:**
1. **Speed** - As you may know, Input Shaping can allow for substantial speed and print quality improvements to your printer. It's all the rage right now in the 3D Printing community.
1. **Low Cost** - It's inexpensive, requiring just $50-100 in hardware, which is a reasonable savings compared to the Prusa MKS3.5 Upgrade Kit.
1. **Simplicity** - We have structured our instructions and documentation to make this as simple as possible, despite the fact that Klipper is involved, and historically has been thought of as difficult to implement. Klipper is very powerful, and can be simple when implemented correctly.
1. **Easy to Return to Stock** - Undoing this upgrade and returning to stock Prusa firmware takes just a few steps, and can be completed in under 10 minutes.
1. **No Waste** - Your MK3S/+ Printer will not collect dust. Many in the Prusa community recommend either buying an MK4 while selling or keeping your MK3S. This upgrade allows you to keep your printer, while substantially increasing speed, and potentially increasing print quality.

**How will we do this?**

We will implement Input Shaping on your MK3S/+ with the use of Klipper. "YIKES, Klipper?! Isn't that super complicated?" No, while Klipper can be difficult to use, we've put in the work for you to allow for a simple  installation and configuration of Klipper. That doesn't necessarily mean it'll be easy, as implementing Klipper's Input Shaping can reveal hardware misconfigurations.

**What if I need to return to stock?**

Undoing this upgrade and returning to stock Prusa firmware can be completed in under 10 minutes. Please see the section "[Reverting to Stock Prusa Firmware](https://github.com/charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade/blob/main/README.md#reverting-to-stock-prusa-firmware)".

## Hardware Required

- Prusa MK3S or MK3S+
  - While configuration files for this project are intended for use with the STOCK hotend and extruder. Other hotends and extruders listed below are possible to use with a modified configuration, but are out of the scope of this repository. For example configurations, please see the dz0ny repository, upon which this project is based: https://github.com/dz0ny/klipper-prusa-mk3s
  - Possible hotends: Stock, Dragon ST, Dragon HF, Revo, many others are supported by Klipper itself.
  - Possible extruders: Stock, Bondtech BMG, many others are supported by Klipper itself.
  - As with Prusa firmware, non-stock extruders require more extensive modification of the configuration.
- Raspberry Pi (one of the following options)
  - Raspberry Pi 3B+: https://amzn.to/3PuZN7D
  - Raspberry Pi 4: https://amzn.to/4a2ZE3p
  - Raspberry Pi Zero 2 W (requires USB adapter in the kit): [https://amzn.to/3ODvStE](https://amzn.to/3TzhrIL)
  - MainSailOS also supports the Orange Pi Zero 2, and the Orange Pi 3 LTS /  4 LTS.
  - Raspberry Pi Zero 1 W is not recommended, as it is underpowered.


- USB Type B male to USB Type A male cable (Came with your Prusa)
- USB Type A female to MicroUSB male converter (if using a Pi Zero 2 W, included in kit linked above)
- An accelerometer (enables easier Input Shaping calibration, optional, but recommended):
  - Direct support in this guide is provided for:
    - KUSBA: Klipper USB Accelerometer (~$20 USD): [https://amzn.to/4bzgAzA](https://amzn.to/49QFR79)
      - NOTE 3/1/2024 - 3D-printed KUSBA mounts online worked but required modification to fit, the nozzle-mounted accelerometer could work better.
    - BTT: BIGTREETECH USB Accelerometer v2.0 (~$16 USD): [https://amazon.com/dp/B0CHJQ6CXF](https://amazon.com/dp/B0CHJQ6CXF)
      - Most easily mounted on the MK3S+
  - Alternative options that may require some configuration:
    - Cheaper alternative: BTT Nozzle-mountable USB Accelerometer v1.0 (~$14 USD): [https://amzn.to/4cg55gQ ](https://amzn.to/3Vgk6t9)
    - Another alternative: FYSETC 3D (~$17 USD) https://amzn.to/3VqvN0B

As an Amazon Associate I earn from qualifying purchases.

## High-level Procedure Overview (see detailed guide below)
**Do not print using PrusaSlicer until Step 6!**

1. Install MainsailOS to your Raspberry Pi
1. Create configuration using the Primary Configuration Files in this repo
1. Flash firmware
    - Use USB method, serial method may only be necessary for some users
1. Perform Config Checks 
1. Customize PrusaSlicer for Klipper
1. Follow Ellis' Guide for primary tuning steps (NOT OPTIONAL)
1. Perform Input Shaper calibrations and measurements - FYI, This step revealed several hardware misconfigurations/issues for me personally as a secondhand printer owner.
1. Re-do Pressure Advance Calibration after enabling IS



# Detailed procedures

## Step 0. Pre-Check and Expectations

- Watch this YouTube video for a basic introduction to Klipper: https://www.youtube.com/watch?v=iNHta6zljoM
- Depending on your familarity with Klipper, expect this full process (including tuning, which is the most time intensive) to take anywhere from 5-20 hours. This range is large because it depends on: 1) your own familarity with tuning procedures that Prusa normally takes care of for you, 2) your familiarity with Klipper, 3) your familiarity with tools like Raspberry Pi, SSH, and command line interfaces. It can also be on the high side if hardware misconfiguration contributes to issues such as bad vibrations in your printer, which will make Input Shaping calibration difficult until addresses. Please be prepared for this.
- Get Z offset value from your current firmware (Menu -> Calibration -> Z-offset), you will need it for the Klipper config.
- Your bed needs to be perpendicular (based on Prusa XYZ Calibration results). An uneven printer means you need to fix the physical hardware first and re-visit the assembly instructions. If not you will have to do the skew calibration before printing or you risk crashing your nozzle to the bed.
- Some of the instructions will be direct links to external guides, which would not make sense to add here, particularly Ellis' 3D Printing guide.

## Step 1. Install MainSailOS on your Raspberry Pi

- We will use MainSailOS as it provides you with all of the components you'll need on your Raspberry Pi. There are other methods, but this is the most simple.
- Please follow this external guide: https://docs-os.mainsail.xyz/

## Step 2. Connect to Raspberry Pi via PC and Configure Klipper

1. Using a web browser, navigate to the URL you selected for your printer, such as `mainsailos.local`.
1. Once loaded, this will be your primary interface for connecting to your Raspberry Pi and Printer in one place. Remember, the Pi will be running the printer just like the Einsy board did on your Prusa.
1. Navigate to Machine. In the Root dropdown, select "config". This is the location of Klipper's critical configuration files.
1. In the Update Manager on the right, update everything. DO NOT UPDATE THESE IN THE FUTURE UNLESS YOU ARE CERTAIN THEY WILL NOT BREAK YOUR CONFIGURATION.
1. In the Config folder, use the files found in this repository's Primary Configuration Files to replace the existing files: https://github.com/charminULTRA/Klipper-Input-Shaping-MK3S-Upgrade/tree/main/1%20-%20Primary%20Configuration%20Files
   - NOTE: Either re-name Printer.Template.cfg, or copy/paste the contents into your own printer.cfg.
1. Edit the printer.cfg as necessary, instructions are inside of the printer.template.cfg file.
1. DO NOT edit the other files unless you know what you are doing.
1. Once all the files have been updated, hit the Power icon at the top right, and select "Restart".

## Step 3. Connect Prusa MK3S/+ physical printer, and flash the Klipper firmware to your printer

1. Plug in the USB Type B male to USB Type A male cable, between your Raspberry Pi and your Prusa printer.
1. We will now be running some commands that will flash the Using SSH (please Google for appropriate methods depending on your PC's operating system), connect to your Raspberry Pi and log in.
1. Run the following commands one by one:
    ```yml
    cd ~/klipper/
    make menuconfig
    ```
Within the configuration menu, the firmware should be compiled for the AVR atmega2560. To use via USB, in "make menuconfig" select "Enable extra low-level configuration options" and select **UART0** when you plan to connect via the USB. Continue by running the following command in order to determine your printer's unique port ID:
```yml
ls /dev/serial/by-id/*
```
After this, run the following one by one. Be sure to update the YOUR_UNQIUE_ID with the printer's unique USB port ID.
```yml
sudo service klipper stop
make flash FLASH_DEVICE=/dev/serial/by-id/YOUR_UNIQUE_ID
sudo service klipper start
```

If this process fails, it is possible that you may need to connect via the Serial port, which is a more involved process, requiring some hardware modification. The Serial port approach is out-of-scope for this guide. Please search Google for the method or reach out on the Klipper Discord: https://discord.com/channels/431557959978450984/1205590905378312293

## Step 4. Perform configuration checks and PID tune

Perform all verify and PID tuning instructions EXCEPT endstops on Klipper's configuration checks page: https://www.klipper3d.org/Config_checks.html
    
## Step 5. Customize PrusaSlicer for Klipper (DO NOT PRINT FROM THE SLICER UNTIL COMPLETING STEP 6!!)
1. Add MK3.5 printer to PrusaSlicer configuration from the Configuration Wizard
1. Copy the MK3.5 printer:
   - Go to Dependencies > "Detach from System Preset"
   - Rename the newly detached printer, you can now use this for your Klipper Printer Profile.
   - Open your new printer. Go to "Printers" and select your new printer from the dropdown on the top left.
   - Set G-code flavor to Klipper. Go to General > Firmware > set "G-code flavor" to "Klipper"
   - Disable binary g-code. Go to General > Firmware > Uncheck "Supports binary G-code".
   - Go to Custom G-code and delete ALL existing data in ALL boxes.
   - Set the start and end code to the contents below. You will not use the factory start and end g-code. The "PRINT_START" and "PRINT_END" commands activate the identically-named macros in your macros.cfg file. Macros are like containers for their own set of Gcode which can be referenced more easily.

        Start Code
        ```yml
        M190 S0 ; Prevents prusaslicer from prepending m190 to the gcode interfering with the macro
        M109 S0 ; Prevents prusaslicer from prepending m109 to the gcode interfering with the macro
        PRINT_START EXTRUDER_TEMP=[first_layer_temperature] BED_TEMP=[first_layer_bed_temperature]
        ```

        End Code
        ```yml
        PRINT_END
        ```
    
   - Save your preset by clicking on the save icon towards the top of the screen.  
1. Copy the factory Prusa printing profiles:
   - For "Print Settings" Presets, and Filament Setting Presets - Using the MK3.5 system presets, go to Dependencies > "Detach from System Preset"
   - Rename the newly detached presets, you can now use this for your Klipper Printer Profile.
   - IMPORTANT: In Print Settings > Advanced - DISABLE "Arc Fitting" - G2/G3 should not be used with Klipper and is not necessary.


## Step 6. Tuning
1. **Prior to printing with PrusaSlicer**, follow Ellis' Guide for primary tuning steps: https://ellis3dp.com/Print-Tuning-Guide/articles/index_tuning.html
   - **Extruder calibration and Pressure Advance are critcal parts of implementing Klipper and cannot be skipped**. Following the other steps are recommended but can be revisited as required.
1. For Pressure Advance and Extrusion Multiplier - It is recommended to add these values in to each individual Filament Profile in PrusaSlicer. PA can be added to the Filaments > Custom G-Code > Start G-code, such as `SET_PRESSURE_ADVANCE ADVANCE=.055`
1. Once you finish tuning, try printing some test prints using the STOCK MK3S+ PROFILES.

## Almost Done!!
You can now print using PrusaSlicer with your found speed. However, this speed comes at the price of worse print quality (ringing) than stock and tuning Input Shaping is the only way to bring it back! It is **highly** encouraged that you continue and tune with an accelerometer. Optionally, print a Benchy now and then run the exact same G-code again after IS tuning to see the difference. If you don't want to or can't get an accelerometer, there are alternatives to the next step that can work well such as https://www.klipper3d.org/Resonance_Compensation.html.

## Step 7. INPUT SHAPING :)
There are many steps, but this doesn't actually take much time and is very automated by Klipper.

To set some expectations, it is important that you run your input shaping tests on a very sturdy surface. Use of foam or any cushioning under your printer is not recommended, as it can skew the results, and you won't be using it after implementing IS anyway. This process, through the resonance measurements, can also reveal potential issues with hardware misconfiguration, such as loose components, loose belts, etc, which can be time consuming to troubleshoot, but will pay off later.

1. Update the firmware on the accelerometer by following the user manual
   - KUSBA: https://github.com/xbst/KUSBA/blob/main/Docs/v2-Firmware.md
   - BTT: https://github.com/bigtreetech/ADXL345
1. Print a toolhead and bed accelerometer mount (if not using the nozzle mount option)
   - KUSBA: https://www.printables.com/search/models?q=kusba&ctx=models
   - BTT v2.0 MK3S+ toolhead: https://www.printables.com/model/1270348-bigtreetech-btt-adxl345-prusa-mk3s-extruder-mount
   - BTT v1.0/v2.0 MK3S/MK3S+ bed: https://www.printables.com/model/1270385-bigtreetech-btt-adxl345-prusa-mk3s-bed-mount
1. Install the accelerometer on the toolhead and connect the USB cable to the Raspbery Pi
1. Uncomment the appropriate accelerometer config file (either adxlmcu-BTT.cfg or adxlmcu-KUSBA.cfg) in your printer.cfg then save and restart the firmware.
1. Verify the accelerometer is sending values
   - Using the console, send the command "ACCELEROMETER_QUERY"
1. Perform X-axis calibration:
   - Using the console, send the command "SHAPER_CALIBRATE AXIS=X"
   - When complete, send the command "SAVE_CONFIG"
1. Remove accelerometer from the toolhead and install onto the bed
1. Perform Y-axis calibration:
   - Using the console, send the command "SHAPER_CALIBRATE AXIS=Y"
   - When complete, send the command "SAVE_CONFIG"
1. Unplug the accelerometer and remove it from the bed. Store it away safely for another day.
1. Open your printer.cfg and comment out (or remove) the accelerometer config file that you enabled in step 5.
1. Save printer.cfg & restart the firmware. You are done tuning IS!

## Step 8. Re-do Pressure Advance after finishing your IS setup
You might get lucky here, but re-running the same test quickly to verify the PA setting is always a good idea.

## THE END! Happy printing.

## Reverting to Stock Prusa Firmware
1. Connect the USB cable from your printer to your PC
1. Download Prusa's firmware from their website: https://help.prusa3d.com/downloads
3. From PruasSlicer, Configuration > Flash Firmware. Select your file and flash.



