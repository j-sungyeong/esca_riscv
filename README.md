# PULPissimo Installation

PULPissimo is a 32-bit single-core platform based on RISC-V core developed by ETH Zurich and University of Bologna. [PULPissimo github](https://github.com/pulp-platform/pulpissimo) is the top-level project where you can find more information. [PULP community](https://pulp-platform.org/community/index.php) is a useful site for Q&A.   
This README is for setting up a working environment of PULPissimo project(with RI5CY core) on FPGA from scratch. It provides the installation process with some instructions for possible errors. Repositories included here __may not be the latest version of PULPissimo project__.

## Getting started
This document is based on the requirements below.   

- Ubuntu 16.04
- Vivado Design Suite 2018.3
- Xilinx ZCU102   
To prevent permission problem, installing everything under home directory is recommended.

## 1. PULP RISC-V GNU Compiler
For more information, please visit here(https://github.com/pulp-platform/pulp-riscv-gnu-toolchain).

### Prerequisites
```
$ sudo apt-get install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev gawk build-essential bison flex texinfo gperf libtool patchutils bc zlib1g-dev
```
### Download sources
This repository uses submodules. Use ```--recursive``` to fetch them.
```
$ git clone --recursive https://github.com/pulp-platform/pulp-riscv-gnu-toolchain
```
### Installation
[TOOLCHAIN_PATH] is where you want to install the compiler. Following command builds 32-bit RISC-V cross-compiler.
```
$ ./configure --prefix=[TOOLCHAIN_PATH] --with-arch=rv32imc --enable-multilib
$ make
```
After build finishes, set ```PULP_RISCV_GCC_TOOLCHAIN``` environment variable.
```
$ export PULP_RISCV_GCC_TOOLCHAIN=[TOOLCHAIN_PATH]
```
## 2. PULP SDK(v1 branch)
For more information, please visit here(https://github.com/pulp-platform/pulp-sdk). v1 branch is out-dated, but this version has some neccesary sources to compile applications for FPGA.
### Prerequisites
```
$ sudo apt install git python3-pip python-pip gawk texinfo libgmp-dev libmpfr-dev libmpc-dev swig3.0 libjpeg-dev lsb-core doxygen python-sphinx sox graphicsmagick-libmagick-dev-compat libsdl2-dev libswitch-perl libftdi1-dev cmake scons libsndfile1-dev
$ sudo pip3 install artifactory twisted prettytable sqlalchemy pyelftools 'openpyxl==2.6.4' xlsxwriter pyyaml numpy configparser pyvcd
$ sudo pip2 install configparser
```
There is an error when upgrading pip to version higher than 21.x.x. To solve this, upgrade pip to version 20.3.4 by ```python3 -m pip install pip=20.3.4``` and ```python -m pip install pip=20.3.4```. There is another solution using python version higher than 3.6 by ```update-alternatives```. After installing packages, pythong version should be changed to 3.5.2 again.

### Download sources
```
$ git clone https://github.com/pulp-platform/pulp-sdk.git -b v1
```

### Build SDK
Be sure to set ```PULP_RISCV_GCC_TOOLCHAIN``` environment variable before building SDK.   
Target platform should be selected first. For FPGA, regardless of your board, source as following and build the SDK:
```
//move to SDK directory
$ source configs/pulpissimo.sh
$ source configs/fpgas/pulpissimo/genesys2.sh
$ make all
```
This SDK has a [gpio input bug](https://github.com/pulp-platform/hal/pull/20/commits/98523f50349f76ebd7e59e5ff95e6869e6a04449). After buiding SDK, ```pkg``` directory will be generated under SDK directory. Fix the bug as following:
```
$ vi pkg/sdk/dev/install/include/hal/gpio/gpio_v3.h

...

//Near line 173
static inline uint32_t hal_gpio_get_value()
{
  return gpio_padout_get(ARCHI_GPIO_ADDR);  //Change this to gpio_padin_get(ARCHI_GPIO_ADDR)
}

```
### Compile Application
Example sources can be found here(https://github.com/pulp-platform/pulp-rt-examples). For example, to build ```hello``` application,   

1. Move to the application directory.
2. Open the application(test.c) and modify the code as following:
```
#include <stdio.h>
#include <rt/rt_api.h>

int __rt_fpga_fc_frequency = 20000000;      //20MHz
int __rt_fpga_periph_frequency = 10000000;  //10MHz

int main()
{
...
}
```
3. Source the SDK
```
$ source pkg/sdk/dev/sourceme.sh
```
4. Compile the application
```
make clean all
```

The elf file 'test' will be generated under ```hello/build/pulpissimo/test```.

### OpenOCD for FPGA
```
//move to SDK directory
$ source sourceme.sh && ./pulp-tools/bin/plpbuild checkout build --p openocd --stdout
```
```OPENOCD``` environment variable is set as ```OPENOCD=[sdk directory]/pkg/openocd/1.0```.

#### Possible error
If you meet something related to 'CMake version' error after sourcing vivado, please launch another terminal without sourcing vivado. There is a conflict regarding it. 

## 3. Porting PULPissimo to FPGA(Xilinx ZCU102)
For more information, please visit here(https://github.com/pulp-platform/pulpissimo). [Vivado 2018.3](https://www.xilinx.com/support/download/index.html/content/xilinx/en/downloadNav/vivado-design-tools/archive.html) should be installed first. Vivado Design Suite license is required for generating bitstream.
```
$ git clone https://github.com/pulp-platform/pulpissimo.git

//move to pulpissimo directory
$ ./update-ips
$ ./generate-scripts

$ source [Vivado install path]/Vivado/2018.3/settings.sh

//move to fpga directory under pulpissimo
$ make zcu102
```
This may take time and needs about 16GB memory. Bitstream file(.bit) will be generated under ```fpga``` directory.   
To download the bitstream to ZCU102, you need micro-USB cable connected to usb-jtag(J2) connector.

1. Run Vivado and click 'Open Hardware Manager'
2. Click 'Open target' - 'Auto Connect'
3. Click 'Program device' and select the .bit file. Then click Program.
4. Type ```set_param xicom.use_bitstream_version_check false``` to the tcl console if you meet 'ERROR: [Common 17-39] 'program_hw_devices' failed due to earlier errors'. This error is due to the board revision.
5. LED0 on the board will be turned on if the bitstream is downloaded well.

### Executing application on the FPGA
After downloading the bitstream, you can load the binary into PULPissimo with OpenOCD and GDB. ZCU102 uses JTAG-HS2 Programming Cable connected to Pmod(j55) to launch OpenOCD. 

1. Download the bitstream to ZCU102.
2. Remove micro-USB cable from the board and close the server from Vivado, or you will meet Error: libusb_claim_interface() failed with LIBUSB_ERROR_BUSY when launching OpenOCD.
3. Connect JTAG-HS2 cable and launch OpenOCD as following:
```
$ $OPENOCD/bin/openocd -f pulpissimo/fpga/openocd-zcu102-digilent-jtag-hs2.cfg
```
4. For UART communication,connect micro-USB cable to usb-UART connector(j83). Open another terminal and launch ```minicom``` as following:
```
$ sudo minicom -s
```
Configure baud rate as 115200 and serial port as /dev/ttyUSB2 among ttyUSB0 ~ ttyUSB3. 

4. In a third terminal, launch GDB with the elf file as following:
```
$PULP_RISCV_GCC_TOOLCHAIN_CI/bin/riscv32-unknown-elf-gdb [PATH_TO_YOUR_ELF_FILE]
```
5. In gdb, run
```
(gdb) target remote localhost:3333
...
(gdb) load
```
Then you can run or debug the application using GDB.

## 4. Simulation with Vivado Simulator
PULPissimo is using Modelsim/Questasim simulator from Mentor Graphics which is not free. Here, simulating on FPGA with Vivado Simulator is provided.