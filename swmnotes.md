# Simon's notes on using the Stratix-10

Once you have a working system booted using the [README.md](README.md), you'll
most likely want to work on new FPGA images wihtout needing to write a
new SDcard.

Note that you may well (still to be tested) need to create a new SD
card if the I/O changes, including external pins or ARM-FPGA bridges.


## Creating SOF file for JTAG download

* Instructions working in:
  de10pro-hps-template/output_files

* Add the boot loader into SOF:
  `quartus_cpf --bootloader=/YourPathTo/de10pro-hps-ubuntu-sdcard-cheri/u-boot-socfpga/spl/u-boot-spl-dtb.ihex DE10_Pro.sof socfpga.sof`

* You can then use `socfpga.sof` to reprogram the FPGA over JTAG

## Create Raw Bit Files (RBF) for boot on the SD card

* Generate rbf files: `socfpga.core.rbf`,  `socfpga.hps.rbf`
  `quartus_cpf -c --hps -o bitstream_compression=on socfpga.sof socfpga.rbf`

* scp the rbf files to /boot on FPGA SD card

* reboot FPGA

* program HPS bit of FPGA:
  `quartus_pgm -m jtag -o P\;./socfpga.hps.rbf@2`
