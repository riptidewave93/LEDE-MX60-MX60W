# LEDE-MX60
Bringup for the Cisco Meraki MX60/MX60W on LEDE!

**NOTE: This repo is NO LONGER MAINTAINED as these changes were merged upstream. Refer to https://github.com/lede-project/source/commit/a72e1692b8b35d61587d75cf1800eef934203cd2 and enjoy the official LEDE nightles!**

Building
-----
#### Build Only
`./build.sh`

#### Modify Configs and Build
`./build.sh modify`

Note that you will need to run a modify on the first compile to select the apm821xx target, MX60/MX60W device in the LEDE menuconfig.

Booting
-----
The MX60/M60W comes with U-Boot, so you can boot an initramfs image using:
```
setenv serverpath; setenv netloadmethod tftpboot; setenv bootargs console=ttyS0,${baudrate} rootfstype=squashfs mtdoops.mtddev=oops; run meraki_load_net meraki_checkpart meraki_bootlinux
```
Note that the file will need to be named `buck.bin` and it will need to be hosted by a TFTP server at 192.168.1.101/24. You will also need to wire into the WAN port of the device, as the LAN ports are not enabled in U-Boot.

Flashing
-----
  1. Boot into U-Boot over UART
  2. Set the new U-Boot settings to boot our images. This will set the proper UBI partition names, disable initramfs loading for the kenel, and keep it for the recovery initramfs image.

  ```
  setenv lede_load1 ubi read \${meraki_loadaddr} kernel

  setenv lede_load2 ubi read \${meraki_loadaddr} recovery

  setenv lede_bootkernel bootm \${meraki_loadaddr_kernel} - \${meraki_loadaddr_fdt}

  setenv lede_bootargs setenv bootargs console=ttyS0,\${baudrate} rootfstype=squashfs mtdoops.mtddev=oops

  setenv lede_boot run meraki_ubi lede_bootargs\; run lede_load1 meraki_checkpart lede_bootkernel\; run lede_load2 meraki_checkpart meraki_bootlinux

  setenv bootcmd run lede_boot

  saveenv
  ```

  3. Use the above Booting commands to boot into an initramfs build of LEDE using TFTP
  4. Once booted, find the UBI Volume ID of board-config. This is done with `ubinfo /dev/ubi0 -N board-config`
  5. Cleanup and move around UBI partitions for maximum space. Note that in this example, replace `XX` with the Volume ID for `board-config`:

  ```
  ubirmvol /dev/ubi0 -N part1
  ubirmvol /dev/ubi0 -N part2
  ubirmvol /dev/ubi0 -N storage
  dd if=/dev/ubi0_XX of=/tmp/board-config.img
  ubirmvol /dev/ubi0 -N board-config
  ubimkvol /dev/ubi0 -s 126KiB -N board-config
  ubiupdatevol /dev/ubi0_0 /tmp/board-config.img
  ```

  6. Create a recovery UBI partition. This will host an initramfs build so our board can have a failback image in case of a bad flash, or sysupgrade issue. Note you will first want to upload a copy of the initramfs image to the board (which can be done with SCP/HTTP Server). In the below tutorial note that the new partition is made to be just a bit larger than the initramfs image. You will want to do this as well.

  ```
  ls -alh /tmp/lede-apm821xx-nand-mx60-initramfs-kernel.bin
  ubimkvol /dev/ubi0 -s 5MiB -N recovery
  ubiupdatevol /dev/ubi0_1 /tmp/lede-apm821xx-nand-mx60-initramfs-kernel.bin
  ```
  7. Once done, you can now load up LuCI at 192.168.1.1, and use the sysupgrade option to flash the full image to the device using the sysupgrade file named `lede-apm821xx-nand-mx60-squashfs-sysupgrade.tar`. From this point on, any future updates/builds can just be flashed through LuCI.

To Do
-----
##### MX60/MX60W
* Nothing

Working
-----
##### MX60/MX60W
* Ethernet
* LED's/GPIOs
* Reset button
* System Integration
* Sysupgrade Support
* Kernel Device Profile
* Kernel Header Generation

Notice
------
No promises this won't brick your unit, and no promises that this will even work!
