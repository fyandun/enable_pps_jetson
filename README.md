### Instructions to get PPS in the GPIO of the Jeston Xavier Dev kit
This is a compilation of instructions to get PPS input on Jetson Boards. They are tested in a Xavier NX Dev Board.

## Jetpack 4.6.4 and lower (Ubuntu 18)
In this case, the configuration is straightforward and can be done after flashing the kernel using the SDK and selecting the default options.

1. Once flashed and in the Ubuntu screen open the terminal.
2. Copy the file src/jetpack4.x/pps.dts in /boot. You will need sudo privilegies for doing that.
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


## Jetpack 5.0.2 and above (Ubuntu 20)
In this case, the process described above did not work. I had to compile the kernel from source and make some modifications found in several forum responses.  I used [this post](https://msadowski.github.io/pps-support-jetson-nano/) as the primary source of information. There is a PDF of this post in the docs folder.

1. Put the board in recovery mode and open the SDK manager. It should be detected automatically.
2. Select the Jetpack to install and click next.
3. At this point we would only need the basic flash option enabled. All the additional components can be skipped and installed later. **Note** At least 50GB will be needed in the host computer to download the sources.
4. When the SDK is ready to flash, cancel and leave the SDK.
5. Download the toolchain. This is required to build the kernel. It changes depending on the Jetpack version (e.g., [Jetson 35.1](https://developer.nvidia.com/embedded/jetson-linux-r351)). 
6. Create a folder to put the compiled kernel: ```mkdir kernel_compiled```
7. Export variables:
```
export CROSS_COMPILE=${PATH_TO_TOOLCHAIN}/bin/aarch64-linux-gnu-
export TEGRA_KERNEL_OUT=${PATH_TO_kernel_compiled}
export LOCALVERSION=-tegra
```
8. Sync the kernel repository by running source_sync.sh in nvidia/nvidia_sdk/JetPack_xx_Linux_xx/Linux_for_Tegra. I used the tag *jetson_35.4.1*.
9. Build the kernel source configuration:
```
cd ${PATH_TO_SDK}/JetPack_XX_Linux_XX/Linux_for_Tegra/sources/kernel/kernel-5.10.
make ARCH=arm64 O=$TEGRA_KERNEL_OUT tegra_defconfig
```
10. Add PPS support by modifying the .config file located in the *kernel_compiled* folder. Modify the pps related lines to look like this:

>\# PPS support
>
>CONFIG_PPS=y
>
>CONFIG_PPS_DEBUG=y
>
>\# PPS clients support
>
>CONFIG_PPS_CLIENT_KTIMER=y
>
>CONFIG_PPS_CLIENT_LDISC=y
>
>CONFIG_PPS_CLIENT_GPIO=m

11. Add the pps support in the device tree code: ${PATH_TO_SDK}/Linux_for_Tegra/sources/hardware/nvidia/soc/tXXX/kernel-dts/tegraXX-soc-base.dtsi. Replace XXX with 194 for the Jetson board and 210 for the Orin. 

The following example corresponds to the Xavier. *TEGRA194_MAIN_GPIO(Q, 5)* comes from the pinmux table, where the pin 29 of the 40 pin header, corresponds to GPIO01 defined as gpio-440 (PQ.05). Check the file *gpio_info.txt* in the folder Jetpack5.x
```
	pps {
		// here use gpio for the pin in which you want pps signal. SYNC_IN
		gpios = <&tegra_main_gpio TEGRA194_MAIN_GPIO(Q, 5) GPIO_ACTIVE_HIGH>; 
		compatible = "pps-gpio";
		assert-falling-edge;
		status = "okay";
	};
```
12. In the file pps-gpio.c (don't remember the location, look for it on the sources within the SDK folder), in the method *pps_gpio_probe*, change this part of the code (line 189 in my case):
```
	/* GPIO setup */
	if (pdata) {
		data->gpio_pin = pdata->gpio_pin;
		data->echo_pin = pdata->echo_pin;

		data->assert_falling_edge = pdata->assert_falling_edge;
		data->capture_clear = pdata->capture_clear;
		data->echo_active_ms = pdata->echo_active_ms;
	} else {
		ret = pps_gpio_setup(pdev);
		if (ret)
			return ret; //  return -EINVAL;
	}
```
This was taken from the [NVIDIA forum](https://forums.developer.nvidia.com/t/agx-xavier-pps-fetch-timeout-r35-2-1-r32-7-1-works-fine/252374). A sample for this file is located in src/jetpack5.x

13. Build the kernel:
```
cd ${PATH_TO_SDK}/Linux_for_Tegra/sources/kernel/kernel-5.1
make ARCH=arm64 O=$TEGRA_KERNEL_OUT -j8
```
14. Prepare the files for flashing the kernel. First, copy and replace the default Image and dtb files, with the ones generated after compilation.
```
cd ${PATH_TO_SDK}/Linux_for_Tegra/kernel
cp $TEGRA_KERNEL_OUT/arch/arm64/boot/Image Image
cd dtb
cp -a /$TEGRA_KERNEL_OUT/arch/arm64/boot/dts/nvidia .
cd ~/nvidia/nvidia_sdk/JetPack_4.3_Linux_P3448/Linux_for_Tegra
```
15. Now we have everything ready to flash the kernel. We have two options here, using the command line or the SDK manager
- command line (jetson Xavier example)
```
sudo ./l4t_create_default_user -u YOUR_USERMAME -p YOUR_PASSWORD
sudo ./flash.sh jetson-xavier-nx-devkit-emmc mmcblk0p1
```
**NOTE**: The -emmc option applies if you want to flash the microSD. If you want to flash the nvme ssd, use the option *jetson-xavier-nx-devkit*.

- SDK
Put the board in recovery mode and don't download anything and just skip to the flashing part. Since you copied and replace the kernel image and device tree files, the SDK will flash the custom kernel.

**NOTE** I prefer the SDK option because it configures the username and password for you. Sometimes using the l4t_create_default_user script, causes the boot to get stuck. 

16. Connect a screen and keboardy/mouse to the board. The first boot usually requires some additional configuration to set up everything. 
17. If everything is fine, pps0 should be present as /dev/pps0
19. Install pps-tools and run tests on pps0

# Useful commands:
- Reading the Kernel configuration: ```zcat /proc/config.gz | grep PPS```
- Reading the gpio_info: ```cat /sys/kernel/debug/gpio```


