# Remote-FPGA-Debug-for-Stratix-10

Setup
Create top folder:
sudo rm -rf stratix10_remotedebug
mkdir stratix10_remotedebug
cd stratix10_remotedebug
export TOP_FOLDER=`pwd`

Download and set up the compiler toolchain, will be used to compile the HPS Debug FSBL:
cd $TOP_FOLDER
wget https://developer.arm.com/-/media/Files/downloads/gnu-a/10.2-2020.11/\
binrel/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz
tar xf gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz
export PATH=`pwd`/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/bin:$PATH
export ARCH=arm64
export CROSS_COMPILE=aarch64-none-linux-gnu-
rm -f gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz

Enable Quartus tools to be called from command line:
export QUARTUS_ROOTDIR=~/intelFPGA_pro/24.1/quartus/
export PATH=$QUARTUS_ROOTDIR/bin:$QUARTUS_ROOTDIR/linux64:$QUARTUS_ROOTDIR/../qsys/bin:$PATH

Build Hardware Design
1. Get the hardware design form github and generate the default configuration:
cd $TOP_FOLDER
rm -rf  agilex_soc_devkit_ghrd  ghrd-socfpga
git clone -b QPDS24.1_REL_GSRD_PR https://github.com/altera-opensource/ghrd-socfpga
mv ghrd-socfpga/s10_soc_devkit_ghrd .
rm -rf  ghrd-socfpga
cd s10_soc_devkit_ghrd
# disable PR to make compilation faster
sed -i 's/ENABLE_PARTIAL_RECONFIGURATION ?= 1/ENABLE_PARTIAL_RECONFIGURATION ?= 0/g' Makefile
make scrub_clean_all
make generate_from_tcl

2. Open the project in Quartus, open the qsys.top file in Platform Designer.

3. In the IP Catalog search for jop and double-click the component to add it to the system:

4. Configure the JOP component as follows:

5. Connect the reset and clock to JOP component, also connect it's slave bus to the HPS LW bridge, and map it at offset 0x0000_8000

6. Finish compilation of the GHRD with Quartus GUI, then generate the sof file:
make -C software/hps_debug
quartus_pfg -c -o hps_path=software/hps_debug/hps_debug.ihex output_files/ghrd_1sx280lu2f50e2vg.sof output_files/ghrd_1sx280lu2f50e2vg_hps_debug.sof
cd ..

Build Core.RBF File
Build the FPGA fabric core configuration file, which will be needed by the Yocto recipes:
cd $TOP_FOLDER
  quartus_pfg -c s10_soc_devkit_ghrd/output_files/ghrd_1sx280lu2f50e2vg_hps_debug.sof \
  ghrd_1sx280lu2f50e2vg.jic \
  -o device=MT25QU128 \
  -o flash_loader=1SX280LU2 \
  -o mode=ASX4 \
  -o hps=1
rm ghrd_1sx280lu2f50e2vg.hps.jic

The following file will be created:
$TOP_FOLDER/ghrd_1sx280lu2f50e2vg.core.rbf

Build Yocto
1. Set up Yocto build:
cd $TOP_FOLDER
rm -rf gsrd_socfpga
git clone -b nanbield https://github.com/altera-opensource/gsrd_socfpga
cd gsrd_socfpga
. stratix10-gsrd-build.sh
build_setup

2. Add the required patch:
wget https://www.rocketboards.org/foswiki/pub/Documentation/RemoteFPGADebugForStratix10/stratix10-jop.patch
pushd meta-intel-fpga-refdes
patch -p1 < ../stratix10-jop.patch
popd
Note: the patch adds the JOP module to the device tree:
diff --git a/recipes-bsp/device-tree/files/socfpga_stratix10_qse_sgmii_ghrd.dtsi b/recipes-bsp/device-tree/files/socfpga_stratix10_qse_sgmii_ghrd.dtsi
 2 index 106efe6..b7de107 100644
 3 --- a/recipes-bsp/device-tree/files/socfpga_stratix10_qse_sgmii_ghrd.dtsi
 4 +++ b/recipes-bsp/device-tree/files/socfpga_stratix10_qse_sgmii_ghrd.dtsi
 5 @@ -46,6 +46,12 @@
 6                                 resetvalue = <0>;
 7                 };
 8  
 9 +               jop@f9008000 {
10 +                               compatible = "generic-uio";
11 +                               reg = <0xf9008000 0x5000>;
12 +                               reg-names = "jop";
13 +               };
14 +
15                 soc_leds: leds {
16                         compatible = "gpio-leds";

   3. Update the recipes to use the compiled core.rbf file created by Quartus:
GHRD_LOC=$WORKSPACE/meta-intel-fpga-refdes/recipes-bsp/ghrd/files
CORE_RBF=$GHRD_LOC/stratix10_gsrd_ghrd.core.rbf
cp $TOP_FOLDER/ghrd_1sx280lu2f50e2vg.core.rbf $CORE_RBF
OLD_CORE_URI="\${GHRD_REPO}\/stratix10_gsrd_\${ARM64_GHRD_CORE_RBF};name=stratix10_gsrd_core"
NEW_CORE_URI="file:\/\/stratix10_gsrd_ghrd.core.rbf"
RECIPE=$WORKSPACE/meta-intel-fpga-refdes/recipes-bsp/ghrd/hw-ref-design.bb
sed -i "s/$OLD_CORE_URI/$NEW_CORE_URI/g" $RECIPE
CORE_SHA=$(sha256sum $CORE_RBF | cut -f1 -d" ")
OLD_CORE_SHA="SRC_URI\[stratix10_gsrd_core\.sha256sum\] = .*"
NEW_CORE_SHA="SRC_URI[stratix10_gsrd_core.sha256sum] =  \"$CORE_SHA\""
sed -i "s/$OLD_CORE_SHA/$NEW_CORE_SHA/g" $RECIPE

4. Build Yocto recipes
bitbake_image

5. Gather the Yocto binaries
package
The bootable SD card image is created as:
$TOP_FOLDER/gsrd_socfpga/stratix10-gsrd-images/gsrd-console-image-stratix10.wic

Build QSPI JIC Image
cd $TOP_FOLDER
  quartus_pfg -c s10_soc_devkit_ghrd/output_files/ghrd_1sx280lu2f50e2vg.sof \
  ghrd_1sx280lu2f50e2vg.jic \
  -o device=MT25QU128 \
  -o flash_loader=1SX280LU2 \
  -o hps_path=gsrd_socfpga/stratix10-gsrd-images/u-boot-stratix10-socdk-gsrd-atf/u-boot-spl-dtb.hex \
  -o mode=ASX4 \
  -o hps=1
The following file will be created:
$TOP_FOLDER/ghrd_1sx280lu2f50e2vg.hps.jic
Run Example
1. Write the bootable image to SD card: $TOP_FOLDER/gsrd_socfpga/stratix10-gsrd-images/gsrd-console-image-stratix10.wic
2. Program the JIC file to QSPI flash: $TOP_FOLDER/ghrd_1sx280lu2f50e2vg.hps.jic
3. Insert the SD card into the board slot, and power cycle the board
4. Wait for Linux to boot, then log in as 'root' with no password:
U-Boot SPL 2023.10 (Mar 21 2024 - 07:41:59 +0000)                                                                                                                                                                                 [695/1813]
Reset state: Cold                                                                                                                                                                                                                           
MPU         1000000 kHz                                                                                                                                                                                                                     
L3 main     400000 kHz                                                                                                                                                                                                                      
Main VCO    2000000 kHz                                                                                                                                                                                                                     
Per VCO     2000000 kHz                                                                                                                                                                                                                     
EOSC1       25000 kHz                                                                                                                                                                                                                       
HPS MMC     50000 kHz                                                                                                                                                                                                                       
UART        100000 kHz                                                                                                                                                                                                                      
DDR: 4096 MiB                                                                                                                                                                                                                               
SDRAM-ECC: Initialized success with 1175 ms                                                                                                                                                                                                 
QSPI: Reference clock at 400000 kHz                                                                                                                                                                                                         
WDT:   Started watchdog@ffd00200 with servicing every 1000ms (10s timeout)                                                                                                                                                                  
denali-nand-dt nand@ffb90000: timeout while waiting for irq 0x2000                                                                                                                                                                          
denali-nand-dt nand@ffb90000: reset not completed.                                                                                                                                                                                          
Trying to boot from MMC1                                                                                                                                                                                                                    
## Checking hash(es) for config board-4 … OK                                                                                                                                                                                              
## Checking hash(es) for Image atf … crc32+ OK                                                                                                                                                                                            
## Checking hash(es) for Image uboot … crc32+ OK                                                                                                                                                                                          
## Checking hash(es) for Image fdt-0 … crc32+ OK                                                                                                                                                                                          
NOTICE:  BL31: v2.10.0  (release):QPDS24.1_REL_GSRD_PR                                                                                                                                                                                      
NOTICE:  BL31: Built : 08:52:03, Mar 21 2024                                                                                                                                                                                                

U-Boot 2023.10 (Mar 21 2024 - 07:41:59 +0000)socfpga_stratix10                                                                                                                                                                              

CPU:   Intel FPGA SoCFPGA Platform (ARMv8 64bit Cortex-A53)                                                                                                                                                                                 
Model: SoCFPGA Stratix 10 SoCDK                                                                                                                                                                                                             
DRAM:  2 GiB (effective 4 GiB)                                                                                                                                                                                                              
Core:  27 devices, 22 uclasses, devicetree: separate                                                                                                                                                                                        
WDT:   Started watchdog@ffd00200 with servicing every 1000ms (10s timeout)                                            
NAND:  denali-nand-dt nand@ffb90000: timeout while waiting for irq 0x2000                                             
denali-nand-dt nand@ffb90000: reset not completed.                                                                    
Failed to initialize Denali NAND controller. (error -5)                                                               
0 MiB                                                      
MMC:   dwmmc0@ff808000: 0                                  
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1...                                             
Loading Environment from UBI... denali-nand-dt nand@ffb90000: timeout while waiting for irq 0x2000                                                                                                                                          
denali-nand-dt nand@ffb90000: reset not completed.                                                                    
SF: Detected mt25qu02g with page size 256 Bytes, erase size 64 KiB, total 256 MiB                                     
Could not find a valid device for ffb90000.nand.0                                                                     
Volume env not found! 

** Unable to read env from root:env **                                                                                                                                                                                                      
In:    serial0@ffc02000                                    
Out:   serial0@ffc02000                                    
Err:   serial0@ffc02000                                    
Net:                                                       
Warning: ethernet@ff800000 (eth0) using random MAC address - fa:dc:cb:26:96:4d                                        
eth0: ethernet@ff800000                                    
Hit any key to stop autoboot:  5  4  3  2  1  0                                                                       
switch to partitions #0, OK                                
mmc0 is current device                                     
Scanning mmc 0:1...                                        
Found U-Boot script /boot.scr.uimg                         
2411 bytes read in 3 ms (784.2 KiB/s)                      
## Executing script at 05ff0000                            
crc32+ Trying to boot Linux from device mmc0                                                                          
Found kernel in mmc0                                       
17198279 bytes read in 813 ms (20.2 MiB/s)                 
## Loading kernel from FIT Image at 01000000 …                                                                      
   Using 'board-4' configuration                           
   Verifying Hash Integrity … OK                         
   Trying 'kernel' kernel subimage                         
     Description:  Linux Kernel                            
     Type:         Kernel Image                            
     Compression:  lzma compressed                         
     Data Start:   0x010000dc                              
     Data Size:    9521379 Bytes = 9.1 MiB                 
     Architecture: AArch64                                 
     OS:           Linux                                   
     Load Address: 0x06000000                              
     Entry Point:  0x06000000                              
     Hash algo:    crc32                                   
     Hash value:   474b072b                                
   Verifying Hash Integrity … crc32+ OK                  
## Loading fdt from FIT Image at 01000000 …                                                                         
   Using 'board-4' configuration                           
   Verifying Hash Integrity … OK                         
   Trying 'fdt-4' fdt subimage                             
     Description:  socfpga_socdk_combined                  
     Type:         Flat Device Tree                        
     Compression:  uncompressed                            
     Data Start:   0x019253b0                              
     Data Size:    37069 Bytes = 36.2 KiB 
Architecture: AArch64                                 
     Hash algo:    crc32                                   
     Hash value:   857c0cd1                                
   Verifying Hash Integrity … crc32+ OK                  
   Booting using the fdt blob at 0x19253b0                 
Working FDT set to 19253b0                                 
## Loading fpga from FIT Image at 01000000 …                                                                        
   Trying 'fpga-4' fpga subimage                           
     Description:  FPGA bitstream for GHRD                 
     Type:         FPGA Image                              
     Compression:  uncompressed                            
     Data Start:   0x01ccd5f0                              
     Data Size:    3772416 Bytes = 3.6 MiB                 
     Load Address: 0x0a000000                              
     Hash algo:    crc32                                   
     Hash value:   831e1d26                                
   Verifying Hash Integrity … crc32+ OK                  
   Loading fpga from 0x01ccd5f0 to 0x0a000000                                                                         
….FPGA reconfiguration OK!                               
Enable FPGA bridges                                        
   Programming full bitstream... OK                        
   Uncompressing Kernel Image                              
   Loading Device Tree to 000000007eac9000, end 000000007ead50cc … OK                                               
Working FDT set to 7eac9000                                
Removing MTD device #2 (root) with use count 1                                                                        
Error when deleting partition "root" (-16)                 
SF: Detected mt25qu02g with page size 256 Bytes, erase size 64 KiB, total 256 MiB                                     
Enabling QSPI at Linux DTB...                              
Working FDT set to 7eac9000                                
libfdt fdt_path_offset() returned FDT_ERR_NOTFOUND                                                                    
libfdt fdt_path_offset() returned FDT_ERR_NOTFOUND                                                                    
QSPI clock frequency updated                               
RSU: Firmware or flash content not supporting RSU                                                                     
RSU: Firmware or flash content not supporting RSU                                                                     
RSU: Firmware or flash content not supporting RSU                                                                     
RSU: Firmware or flash content not supporting RSU                                                                     

Starting kernel …                             

Deasserting all peripheral resets                          
[    0.000000] Booting Linux on physical CPU 0x0000000000 [0x410fd034]                                                
[    0.000000] Linux version 6.1.68-altera (oe-user@oe-host) (aarch64-poky-linux-gcc (GCC) 13.2.0, GNU ld (GNU Binutils) 2.41.0.20231213) #1 SMP PREEMPT Thu Mar 28 07:56:27 UTC 2024
[    0.000000] Machine model: SoCFPGA Stratix 10 SoCDK

…

[  OK  ] Started Hostname Service.                         
[   15.077323] socfpga-dwmac ff800000.ethernet eth0: Link is Up - 1Gbps/Full - flow control rx/tx                                                                                                                                           
[   15.086005] IPv6: ADDRCONF(NETDEV_CHANGE): eth0: link becomes ready                                                

Poky (Yocto Project Reference Distro) 4.3.4 stratix10 ttyS0                                                           

stratix10 login: root
root@stratix10:~#

4. Load the UIO driver:
root@stratix10:~# rmmod uio_pdrv_genirq
root@stratix10:~# modprobe uio_pdrv_genirq of_id="generic-uio"

5. Run ifconfig to determine the IP address of your board:
root@stratix10:~# ifconfig
eth0: flags=-28605<UP,BROADCAST,RUNNING,MULTICAST,DYNAMIC>  mtu 1500
        inet 192.168.1.142  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::f86b:cfff:feec:5a5  prefixlen 64  scopeid 0x20
        ether fa:6b:cf:ec:05:a5  txqueuelen 1000  (Ethernet)
        RX packets 62  bytes 5243 (5.1 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 62  bytes 9902 (9.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        device interrupt 21  base 0x2000

   6. Run Etherlink, specifying the port number, otherwise it would be random port assigned:
root@stratix10:~# etherlink --port=33301
INFO: Etherlink Server Configuration:
INFO:    H2T/T2H Memory Size  : 4096
INFO:    Listening Port       : 33301
INFO:    IP Address           : 0.0.0.0
INFO: UIO Platform Configuration:
INFO:    Driver Path: /dev/uio0
INFO:    Address Span: 20480
INFO:    Start Address: 0x0
INFO: Server socket is listening on port: 33301

7. On the host PC, set up the remote debug link to the board
jtagconfig --add JTAG-over-protocol sti://localhost:0/intel/remote-debug/192.168.1.142:33301/0

8. On the host PC, run jtagconfig to show the new available remote JTAG connection:
jtagconfig
1) JTAG-over-protocol [sti://localhost:0/intel/remote-debug/192.168.1.142:33301/0]
  020D10DD   VTAP10

2) Stratix 10L SoC Dev Kit [1-3.4.4]
  6BA00477   S10HPS/AGILEX_HPS/N5X_HPS
  C321D0DD   1SX280LH(2|3)/1SX280LN2(|AS)/..
The above connection will be available to tools like SignalTap, enabling remote debug scenarios.
