Getting started with the Terasic DE10-Pro FPGA Board
----------------------------------------------------

The following instructions are a (barebones) guide for getting started with
the DE10 Pro.

They assume an x86_64 machine running a recent Ubuntu LTS (both 18.04 and 20.04
should work).  A virtual machine is acceptable although you will have to do
some USB passthrough for JTAG programming.


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

A file sdimage.img should be generated which is suitable for writing to an
SD card via dd, Etcher or [USBImager](https://gitlab.com/bztsrc/usbimager)).

If something goes wrong, a log is produced in sdbuild.log.
