### Instructions to get PPS in the GPIO of the Jeston Xavier Dev kit
This is a compilation of instructions to get PPS input on Jetson Boards. They are tested in a Xavier NX Dev Board.

## Jetpack 4.6.4 and lower (Ubuntu 18)
In this case, the configuration is straightforward and can be done after flashing the kernel using the SDK and selecting the default options.

1. Once flashed and in the Ubuntu screen open the terminal.
2. Copy the file pps.dts in /boot. You will need sudo privilegies for doing that.
3. Compile the dts file doing ```sudo dtc -I dts -O dtb -@ -o pps.dtbo pps.dts```
4. Check the board can read the configuration: ``` sudo /opt/nvidia/jetson-io/config-by-hardware.py -l ```. You should see something like this:
    
    Configurations for the following hardware modules are available:
    1. Adafruit SPH0645LM4H
    2. FE-PI Audio Z V2
    3. Jetson PPS
    **
5. Apply that configuration: ```sudo /opt/nvidia/jetson-io/config-by-hardware.py -n "Jetson PPS"```
6. Reboot
7. After rebooting, check the file /boot/extlinux/extlinux.conf. Below the block entitled LABEL backup you should see another -uncommented- block entitled **LABEL Jetson PPS**
8. If everything is fine, pps0 should be present as /dev/pps0
9. Install pps-tools and run tests on pps0

# Notes
In the file pps.dts line 14, the number 133 was obtained from the [jetson hacks website](https://jetsonhacks.com/nvidia-jetson-xavier-nx-gpio-header-pinout/). My guess it is calculated based on the gpio info. Pin 29 is GPIO01 corresponds to gpio421, which has an offset of 133 from the base (162-29 -> these are line numbers). **IMPORTANT:** These values changed in Jetpack 5.X, which made more difficult to apply the exact same thing.


## Jetpack 4.6.4 and lower (Ubuntu 18)


# Useful commands:
- Reading the Kernel configuration: ```zcat /proc/config.gz | grep PPS```
- Reading the gpio_info: ```cat /sys/kernel/debug/gpio```


