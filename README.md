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
of Quartus to another is *not straightforward* and you should think
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
# may need to unplug and replug FPGA USB cable if attached
```

Running Quartus
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
version of Quartus you want (multiple can be installed).  This only works in
bash, not other shells:

```
waltham:$ source /local/ecad/setup.bash 19.2pro
```

