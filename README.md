Getting started with the Terasic DE10-Pro FPGA Board
----------------------------------------------------

The following instructions are a (barebones) guide for getting started with
the DE10 Pro.

They assume an x86_64 machine running a recent Ubuntu LTS (both 18.04 and 20.04
should work).  A virtual machine is acceptable although you will have to do
some USB passthrough for JTAG programming.

You will need about 62GiB free space in total, although you can recover
about 20GiB by deleting the Quartus installer files.  It requires about
30GiB of downloads.

Install Quartus Pro
-------------------

The current recommended version is 19.2 Pro (this is different from the
'Lite' or 'Standard' versions).  Please note that porting from one version
of Quartus to another is **not straightforward** and you should think
carefully before building with a different version.

```
sudo apt install aria2
git clone https://github.com/CTSRD-CHERI/quartus-install.git
sudo mkdir -p /local/ecad/altera
sudo chown -R $USER:$USER /local/ecad/altera
cd quartus-install
./quartus-install.py --fix-libpng 19.2pro /local/ecad/altera/19.2pro s10
# wait while 23GiB downloads and installs
```

Meanwhile, configure USB JTAG permissions:

```
cat <<EOF | sudo tee /etc/udev/rules.d/51-usbblaster.rules
# USB-Blaster
SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6001", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6002", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6003", MODE="0666", GROUP="plugdev"

# USB-Blaster II
SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6010", MODE="0666", GROUP="plugdev"
SUBSYSTEM=="usb", ATTR{idVendor}=="09fb", ATTR{idProduct}=="6810", MODE="0666", GROUP="plugdev"
EOF

sudo udevadm control --reload-rules
```

You may need to unplug and replug FPGA USB cable if it is already attached
when the rules are reloaded.

**Virtual machine**: If running Ubuntu in a VM (eg VirtualBox), in your
hypervisor config create mappings to passthrough USB devices with the vendor
IDs and product IDs listed above - you will probably only need those in the
USB Blaster II section.


Quartus licensing
---------------

Whenever you run Quartus, you will need to provide it with a licence server:
```
export LM_LICENSE_FILE=27012@lmserv-altera.cl.cam.ac.uk
```

That should work if you're on VPN, but if not you can SSH tunnel:

```
ssh -L 27012:lmserv-altera.cl.cam.ac.uk:27012 \
    -L 27013:lmserv-altera.cl.cam.ac.uk:27013 \
    -f waltham.cl.cam.ac.uk sleep 999999
export LM_LICENSE_FILE=27012@localhost
```

You will also need to add it to your PATH:
```
export QUARTUS_ROOTDIR=/local/ecad/altera/19.2pro/quartus
export PATH=$QUARTUS_ROOTDIR/bin:$PATH
```

On CL servers these are combined in a script, which takes as a parameter the
version of Quartus you want (multiple may be installed).  This only works in
bash, not other shells:

```
waltham:$ source /local/ecad/setup.bash 19.2pro
```

Running Quartus
---------------

To run Quartus GUI, enter:

```
quartus &
```

After a short while, the large Quartus window should pop up.  If you then go
to Tools -> License Setup you should see a long list of licences provided by
the licence setup.  (If you are prompted about installing devices, or asked
what to do about licences, something went wrong in your installation or
licence config respectively).

It is now safe to remove your quartus-install tree to save about 23GiB.


ARM (HPS) template project for the DE10-Pro SX
----------------------------------------------

To obtain and build the ARM-side template project for the DE10Pro:

```
git clone https://github.com/CTSRD-CHERI/de10pro-hps-template.git
```

In Quartus GUI, File->Open Project, select de10pro-hps-template/DE10_Pro.qpf.

Press the 'Play' button on the toolbar, or Processing->Start Compliation, to
compile.  Compilations should take about 20 minutes.


Building the Ubuntu SD card image
---------------------------------

When the Stratix 10 is set to boot ARM-side MMC-first (rather than FPGA-side
first), we need to provide an SD card image with the following components

* FPGA HPS configuration bitfile
* FPGA I/O partial configuration bitfile (enables ARM access to DDR4 RAM)
* U-boot bootloader
* U-boot Device Tree
* Linux kernel
* Linux Device Tree
* Linux initramfs
* Linux root filesystem

This is built in the following way:

```
sudo apt install libncurses-dev libssl-dev device-tree-compiler
git clone https://github.com/CTSRD-CHERI/de10pro-hps-ubuntu-sdcard-cheri.git
cd de10pro-hps-ubuntu-sdcard-cheri
git submodule init
git submodule update
mkdir payload
# this merges files from the tree in 'payload' into the image and installs the
# packages vim-tiny and openssh-server
./scripts/build_ubuntu_sdcard.sh \
  ../de10pro-hps-template DE10_Pro DE10_Pro_HPS payload \
  vim-tiny openssh-server  \
  | tee sdbuild.log
```

When the Linux config menu appears, just cursor to Exit and press Enter. 
Then press Enter to confirm no device tree edits for Linux, and later again
for U-boot.  

A file sdimage.img should be generated which is suitable for writing to a
Micro SD card via dd, Etcher or [USBImager](https://gitlab.com/bztsrc/usbimager)).

If something goes wrong, a log is produced in sdbuild.log.


Booting the DE10
----------------

Attach power to the DE10 board.  Attach a cable to the Mini USB port on the
Terasic HPS daughterboard.  Fit the SD card into the slot on the rear of the
DE10.  Check the DIP switches on the rear are set as below:

FIXME: DIP switch image

Run a serial console
```
sudo apt install picocom
sudo usermod -a -G dialout $USER
sudo picocom -b 115200 /dev/ttyUSB0
```

(the group-add won't take effect until you next logout - after that you can
drop the `sudo` before `picocom`)

Watch as the DE10 boots.  You should get to a login prompt.
**Wait!** On first boot user account provision doesn't come through until cloud-init
starts, which might be up to a minute later.  Wait until cloud-init displays some
messages, like:

```
fpga login: 
[   50.854071] cloud-init[2039]: Cloud-init v. 20.2-45-g5f7825e2-0ubuntu1~18.04.1 running 'modules:config' at Sun, 28 Jan 2018 15:59:03 +0000. Up 48.15 seconds.
...
[   55.026003] cloud-init[2074]: Cloud-init v. 20.2-45-g5f7825e2-0ubuntu1~18.04.1 finished at Sun, 28 Jan 2018 15:59:10 +0000. Datasource DataSourceNoCloud [seed=/dev/mmcblk0p1][dsmode=net].  Up 54.97 seconds
```

Now you can login with username/password `ubuntu`/`ubuntu`.  You'll be prompted
to change your password.  Next time you boot you can login with this
password - you don't need to wait for cloud-init.


FPGA boot bitfile configuration
--------------------------

The HPS-first FPGA boot process goes like this:

- A bitfile for HPS I/Os is loaded from QSPI flash.  This is enough to
  configure the SD/MMC controller, HPS ethernet and HPS USB
- A U-boot second stage bootloader is embedded in this bitfile, which loads the
remainder of U-boot from the SD card. U-boot loads a Device Tree file (for itself)
- U-boot has a .scr script file describing what to do next
- In our image, we mount the SD/MMC card. We then partially configure the FPGA with
the bitfile we built above, which connects a DDR4 memory controller to the ARM
- Then we load a Linux kernel, Linux Device Tree, and initramfs
- We then boot Linux
- At a later time, Linux can use the [FPGA Manager
API](https://www.kernel.org/doc/html/latest/driver-api/fpga/fpga-mgr.html)
to partially reconfigure further bitfiles (and their Device Trees to
configure Linux appropriately).  In the case of the Stratix 10
the Linux kernel makes a call to the still-resident U-boot firmware which does the
reprogramming.

The SD card contains the FPGA+ARM bitfile we built above, which gets loaded
into the FPGA by U-boot.  However, being a partial reconfiguration bitfile,
it depends on a matching bitfile being present in QSPI flash.

If your DE10 is unmodified, the bitfile in QSPI won't match the part we
built, and the FPGA configuration by U-boot will fail (but boot will
continue anyway and succeed):
```
U-Boot 2017.09-00157-gdec0cf16d1 (Jan 26 2021 - 12:03:43 +0000)socfpga_stratix10

CPU:   Intel FPGA SoCFPGA Platform
FPGA:  Intel FPGA Stratix 10
Model: Terasic DE10-Pro
DRAM:  2 GiB
MMC:   dwmmc0@0xff808000: 0
*** Warning - bad CRC, using default environment

In:    serial
Out:   serial
Err:   serial
Model: Terasic DE10-Pro
Net:   No ethernet found.
Hit any key to stop autoboot:  0 
reading u-boot.scr
313 bytes read in 3 ms (101.6 KiB/s)
## Executing script at 02100000
reading socfpga.core.rbf
1757184 bytes read in 119 ms (14.1 MiB/s)
..RECONFIG_DATA error: 00000002, Unknown error!
```

To resolve this problem without the QSPI being reflashed, we need to override
the QSPI bitfile by programming the DE10 via JTAG.  This can be done by
connecting a Micro USB cable to the port on the DE10 rear bracket (not the
HPS daughterboard).  To confim a successful JTAG connection, run
`jtagconfig` - you should see:

```
$ jtagconfig
1) DE10-Pro [1-13.2]
  6BA00477   S10HPS
  C322D0DD   1SX280HH1(.|S3)/1SX280HH2/..
```

To program the FPGA, run:
```
quartus_pgm -m jtag -o P\;de10pro-hps-template/output_files/DE10_Pro-hps.hps.rbf@2
```

When the FPGA comes out of reset it will start the SD card.  You should see
U-boot then correctly reprograms the FPGA:

```
Hit any key to stop autoboot:  0 
reading u-boot.scr
313 bytes read in 3 ms (101.6 KiB/s)
## Executing script at 02100000
reading socfpga.core.rbf
1757184 bytes read in 118 ms (14.2 MiB/s)
```
