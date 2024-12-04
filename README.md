Simplified setup for Waveshare 3.5 inch LCD (A) for Raspberry Pi OS bookworm
============================================================================

This repo summarises how to get the Waveshare 3.5 inch LCD (type A) working with Raspberry Pi OS bookworm (tested the 64 bit version on a Raspberry 
Pi 3).  The official Waveshare setup script is old, quite invasive to your system (overwriting some settings) and not fully 64 bit friendly.

The Waveshare screen uses the ILI9486 SPI display controller and ADS7846 SPI resistive touch controller.  These instructions may work for clones that 
use compatible controllers - this is untested.

Since we'll be reconfiguring the Pi's video system, I suggest doing the following changes via SSH, not using the HDMI port, as they may disable the HDMI output.


1. Connect your LCD to the Pi's GPIO header.  The Pi has a 40 pin header, the display has 26 pins.  You want the end of the row furthest from the USB 
sockets.  You may need a riser header if the Pi's own header isn't tall enough to clear the USBs.

2. Boot the Pi and copy the device tree overlay file by running the following:

   ```
   git clone https://github.com/caliston/waveshare-LCD35-bookworm
   cd waveshare-LCD35-bookworm
   sudo cp waveshare35a-overlay.dtb /boot/overlays/waveshare35a.dtbo
   ```

   This file tells Linux about the display hardware we've connected.

3. We need to tell the Pi the existance of the .dtbo file.  Edit /boot/config.txt and add to the bottom:
(November 2024: This file is now located at /boot/firmware/config.txt in the latest bullseye images)

   ```
   dtoverlay=waveshare35a
   hdmi_force_hotplug=1
   hdmi_group=2
   hdmi_mode=1
   hdmi_mode=87
   hdmi_cvt 480 320 60 6 0 0 0
   hdmi_drive=2
   ```

   While in the file, make sure the following are present and not commented out (add them if necessary):

   ```
   dtparam=i2c_arm=on
   dtparam=spi=on
   disable_fw_kms_setup=1
   ```

   and comment out the following entries:

   ```
   #dtoverlay=vc4-kms-v3d
   #max_framebuffers=2
   ```

   Save the file.  This enables SPI, sets the video mode to 480x320, and disables the native framebuffer support.
   (this may cause the HDMI output to stop working.

4. Reboot the Pi:

   ```
   sudo shutdown -r now
   ```

   Once the Pi has come back, run `dmesg`.  As it boots the display should turn grey.  You should see some relevant log lines of setting up the LCD and the touch controller:

   ```
   [   12.075247] ads7846 spi0.1: supply vcc not found, using dummy regulator
   [   12.083302] ads7846 spi0.1: touchscreen, irq 185
   [   12.089305] input: ADS7846 Touchscreen as /devices/platform/soc/3f204000.spi/spi_master/spi0/spi0.1/input/input0
   ...
   [   12.420322] fb_ili9486: module is from the staging directory, the quality is unknown, you have been warned.
   [   12.421084] SPI driver fb_ili9486 has no spi_device_id for ilitek,ili9486
   [   12.438820] fb_ili9486 spi0.0: fbtft_property_value: regwidth = 16
   [   12.438866] fb_ili9486 spi0.0: fbtft_property_value: buswidth = 8
   [   12.438902] fb_ili9486 spi0.0: fbtft_property_value: debug = 0
   [   12.438928] fb_ili9486 spi0.0: fbtft_property_value: rotate = 90
   [   12.438958] fb_ili9486 spi0.0: fbtft_property_value: fps = 30
   [   12.438984] fb_ili9486 spi0.0: fbtft_property_value: txbuflen = 32768
   ...
   [   13.044785] graphics fb1: fb_ili9486 frame buffer, 480x320, 300 KiB video memory, 32 KiB buffer memory, fps=31, spi0.0 at 16 MHz
   ```

   You should also see a /dev/fb0 or /dev/fb1 device appear:

   ```
   $ ls -l /dev/fb*
   crw--w---- 1 root video 29, 0 Jan 15 22:56 /dev/fb0
   crw--w---- 1 root video 29, 1 Jan 16 16:54 /dev/fb1
   ```

   You can test out this node by doing the following:

   ```
   sudo dd if=/dev/urandom of=/dev/fb1
   ```

   If you see multicoloured 'snow' on a part of the display you know it's working.

5. With older images the permissions on this node aren't right so we need a udev rule to fix it.  Put this in /etc/udev/rules.d/99-tty.rules if the permissions on the /dev/fb* devices above are `crw--w----` :

   ```
   KERNEL=="tty[0-9]*", GROUP="tty", MODE="0660"
   ```

   You will also need to add your user to the 'video' user if not already:

   ```
   sudo usermod -a -G video $USER
   ```

6. We also need to configure X11 to use this display.  Put this in /etc/X11/xorg.conf.d/99-fbdev.conf :

   ```
   Section "Device"
           Identifier      "Allwinner A10/A13 FBDEV"
           Driver          "fbdev"
           Option          "fbdev" "/dev/fb1"
           Option          "SwapbuffersWait" "true"
   EndSection
   ```

7. Reboot the Pi.  With luck you will see the windowing system (or another X11 app like Klipperscreen) appear on the display.

8. The touch input should work, but you may find its input is rotated so it's clicking on the wrong things.  Edit /usr/share/X11/xorg.conf.d/40-libinput.conf and add a 
CalibrationMatrix line as follows:

   ```
   Section "InputClass"
           Identifier "libinput touchscreen catchall"
           MatchIsTouchscreen "on"
           MatchDevicePath "/dev/input/event*"
           Driver "libinput"
           Option "CalibrationMatrix" "0 -1 1 1 0 0 0 0 1"
   EndSection
   ```

   Alternative CalibrationMatrix lines for different orientations:

   - 0 degrees: `Option "CalibrationMatrix" "1 0 0 0 1 0 0 0 1"`
   - 90 degrees: `Option "CalibrationMatrix" "0 1 0 -1 0 1 0 0 1"`
   - 180 degrees: `Option "CalibrationMatrix" "-1 0 1 0 -1 1 0 0 1"`
   - 270 degrees: `Option "CalibrationMatrix" "0 -1 1 1 0 0 0 0 1"`

9. To get a text console on the display you can add the following to /boot/firmware/cmdline.txt (/boot/cmdline.txt with older images): ` fbcon=map:10`.
This will map the tty1 console to the display; there are other options you can supply to set rotation and font. Swapping to a 16x8 font from the default aids readability.
A full cmdline.txt is:
   ```
   console=serial0,115200 console=tty1 root=PARTUUID=52157a78-02 rootfstype=ext4 fsck.repair=yes rootwait cfg80211.ieee80211_regdom=NL fbcon=map:10 fbcon=rotate:0 fbcon=font:VGA8x16
   ```
for more see: https://www.kernel.org/doc/html/latest/fb/fbcon.html
