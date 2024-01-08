# Deployment Guide for Oai CU - Osc Odu High - Oai L1

# Basic Info
**[supermicro Server]**
⚓ **Installation environment:**
- Branch: `Develop`
- OS: Ubuntu 20.04.6 LTS (Focal Fossa) 
- CPU: Intel(R) Xeon(R) Gold 6312U CPU @ 2.40GHz core 48
- Stroge: 1Tb
- Memory: 256GB

## File Downloads and refernces 
| Drive | `Ngkore > hackmd > Oai-CU-Osc-L2-Oai-L1`|
|-|-|


## Basic requirement
| OS/Kernel                  | Version                                 |
|:-------------------------- |:----------------------------------------|
| Host operating system      | Ubuntu 20.04.6 LTS (Focal Fossa)     |
| Kernel                     | 5.4.0-169-lowlatency                   |


# 1.  Setup server environment

Update newest package.
```bash
sudo apt-get update -y
```

## 1.1 Change Kernel

**Goal:** `5.4.0-169-lowlatency`

Download target kernel
```bash
sudo apt-get install linux-image-5.4.0-169-lowlatency
```

Check OS indtalled kernel
```bash
dpkg --list | grep linux-image
```

Check kernel list
```bash
grep -E "(submenu|menuentry) " /boot/grub/grub.cfg
```
> check submenu 'Advanced options for Ubuntu' ID
> check menuentry 'Ubuntu, with Linux 5.4.0-169-lowlatency'

![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/10fb9ecb-af11-4d95-ba24-2b529e51e0b2)

Edit file
```bash
sudo nano /etc/default/grub
```
- Find `GRUB_DEFAULT=1` and changes into 
```bash!
GRUB_DEFAULT="gnulinux-advanced-02051e5b-e6dc-4364-9531-869c95a70f94>gnulinux-5.4.0-169-lowlatency-advanced-02051e5b-e6dc-4364-9531-869c95a70f94"
```
> You need to change to your ID


Update GRUB
```bash
sudo update-grub
```

Change reboot default kernel
```bash
sudo grub-set-default '5.4.0-169-lowlatency'
```

Manual **Reboot**

Check current OS kernel
```bash
uname -r
5.4.0-169-lowlatency
```

## 1.2 Install UHD （USRP Hardware Driver）

**Reference:**
-  [Set up UHD_USRP](https://hackmd.io/@MingHung/UHD_USRP)

Intstll package
```bash
sudo apt-get install libuhd-dev
```
Try to find your USRP
```bash
uhd_find_devices
```
Successful result
```bash
$ sudo uhd_find_devices
[INFO] [UHD] linux; GNU C++ version 7.5.0; Boost_106501; UHD_4.3.0.0-0ubuntu1~bionic1
--------------------------------------------------
-- UHD Device 0
--------------------------------------------------
Device Address:
    serial: 31502CB
    name: MyB210
    product: B210
    type: b200
```

```bash=
sudo updatedb
locate uhd_images_downloader.py

# run this python file
sudo /usr/lib/uhd/utils/uhd_images_downloader.py
```

<details>
  <summary>Problem 1</summary>
  
```
uhd_find_devices: error while loading shared libraries: libuhd.so.4.6.0: cannot open shared object file: No such file or directory
```
```
ldconfig -p |grep uhd
```
![image](https://hackmd.io/_uploads/rJLt_cK8p.png)
 
```bash
wget https://github.com/EttusResearch/uhd/releases/download/v4.5.0.0/uhd-images_4.5.0.0.zip
```

</details>

## 1.3 Linux Output the timestamp
```bash=
apt install moreutils
```
[csdn link](https://blog.csdn.net/m0_38059875/article/details/115401310)

# 2. Building OAI CU

```shell
cd oscdu_oaicu/openairinterface5g/cmake_targets

## Build Command
./build_oai --gNB
```

# 3. Building OAI L1 (USRP)

```bash=
cd openairinterface5g

# Build ONLY FOR the FIRST time
source oaienv
cd cmake_targets
export BUILD_UHD_FROM_SOURCE=Ture
export UHD_VERSION=4.5.0

# export UHD_VERSION=4.3.0.0-rc1
# we upgrade to latest version 4.5.0

./build_oai -I -c -C
./build_oai -w USRP --gNB
```

# 4.  Prerequisite for O-DU High
Install necessary libraries to build & compile O-DU high.

```bash
## GCC, make sure your GCC version is 4.6.3 or above for compiling, and install it if necessary.
gcc --version
## Install GCC by below command
sudo yum groups mark install -y "Development Tools"
## Ubuntu : sudo apt-get install -y build-essential

## LKSCTP
sudo yum install -y lksctp-tools-devel
## Ubuntu : sudo apt-get install -y libsctp-dev

## PCAP 
sudo yum install -y libpcap-devel
## Ubuntu : sudo apt-get install -y libpcap-dev
```

# 5.  Building O-DU and RIC Stub binary
```bash=
cd ~/OAIOSC/OSC_L2/build/odu

# O-DU High
make clean_odu MACHINE=BIT64 MODE=FDD VNF_ENABLE=YES
make odu MACHINE=BIT64 MODE=FDD VNF_ENABLE=YES
make odu MACHINE=BIT64 MODE=FDD VNF_ENABLE=YES O1_ENABLE=YES

# RIC Stub
make clean_ric NODE=TEST_STUB MACHINE=BIT64 MODE=FDD
make ric_stub NODE=TEST_STUB MACHINE=BIT64 MODE=FDD
make ric_stub NODE=TEST_STUB MACHINE=BIT64 MODE=FDD O1_ENABLE=YES
```

# 6. Assign IP address for O-DU and OAI CU

:::danger
After the **initial installation** or each **reboot**, 
remember to perform this step ++before executing gNB++.
:::
:::warning
If you can't use `ifconfig`, run below command to install tools.
:::
```shell=
sudo apt install net-tools -y
```

script:
```
#!/bin/bash
echo "***** Assigning IP addresses *****"
INTERFACE=$(ip route | grep default | sed -e "s/^.*dev.//" -e "s/.proto.*//")
INTERFACE="$(echo -e "${INTERFACE}" | tr -d '[:space:]')"

sudo ifconfig $INTERFACE:ODU  "192.168.130.81"
sudo ifconfig $INTERFACE:CU_STUB "192.168.130.82"
sudo ifconfig $INTERFACE:RIC_STUB "192.168.130.80"
sudo ifconfig $INTERFACE:OAI_CU "192.168.130.83"

PATH=/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin:/usr/local/sbin:~/home/gnb/
export PATH
echo  "O-RAN NIC Setting Complete!"
```

# 7. Execution

## Run Oai CU
```bash
 sudo ./nr-softmodem -O ../../../ci-scripts/conf_files/cu.band66.tm1.106PRB.usrpb210.conf --sa
```
under
```
/home/six/oscdu_oaicu/openairinterface5g/cmake_targets/ran_build/build
```

## Run Osc L2
```
./odu
```
under
```
/home/six/NCU_integration_OSC_DU-main/bin/odu# 
```

## Run Oai L1
```bash
sudo ./nr-softmodem -O ../../../../NCU_integration_OSC_DU-main/mwnl/oai_pnf_conf/oaiL1.nfapi.usrpb210.conf --sa -E --gNBs.[0].min_rxtxtime 6 --continuous-tx --nfapi 1
```
under
```
/home/six/openairinterface5g/cmake_targets/ran_build/build#
```

<details>
  <summary>Extras for further ref.</summary>

**Or you can execute our [scrip](https://github.com/Alt-Shivam/run-integrate_OAIOSC) to run integration testing**
Next, you can check [the logs we've gathered on our GitHub](https://github.com/Alt-Shivam/OSC-DUHigh_OAI-L1).


```bash
git clone https://github.com/Alt-Shivam/run-integrate_OAIOSC
```

If you reboot the server, have to execute [nic.sh](https://github.com/Alt-Shivam/run-integrate_OAIOSC) to assign IP address for O-DU and OAI CU

![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/3d7a2297-c824-49b2-a0f7-26ac5b1cd57f)



# ⚠️  Issue

## 2. B200 operating over USB 2.

```bash!
[WARNING] [B200] The recv_frame_size must be a multiple of 8 bytes and not a multiple of 512 bytes. Requested recv_frame_size of 7680 coerced to 7688.
B200 operating over USB 2.
[WARNING] [MULTI_USRP] The total sum of rates (46.080000 MSps on 1 channels) exceeds the maximum capacity of the connection. This can cause overflows (O).
[WARNING] [MULTI_USRP] The total sum of rates (46.080000 MSps on 1 channels)
```
![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/17bf1609-9389-49a5-9105-88ddcd077476)


### success
The USRP requires the use of USB 3.0

## 3. can't open the radio device: none
The issue arises when we initially execute OAI L1.
![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/d3ec93ad-e13a-4041-828d-4cd1608be16f)

Whenever we restart the server, we have to unplug and plug in the USRP again.
![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/2b556133-3d11-40cd-817f-433e57e537bc)


![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/073df25a-4abe-4621-bcc6-185d44af5093)


### success
We found out the solution is to restart the USB port.
```bash
sudo service udev restart
```
:::

## 4. The PNF config's mcc and mnc is NOT same as OSC L2
:::info
https://hackmd.io/@MingHung/H10J_wm7a
:::
:::success
This has been solved in MR 2304, merged in tag [2023.w38. ](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/2023.w38?ref_type=tags)Please upgrade to that, or preferably newer.
:::



## 5. Can't build new version OAI PNF [2023.w38. ](https://gitlab.eurecom.fr/oai/openairinterface5g/-/tree/2023.w38?ref_type=tags)

```bash=
./build_oai -w USRP --gNB
```

Excerpt of error message at the end of running the above command:

```bash
Running "cmake --build .  --target oai_usrpdevif nr-softmodem nr-cuup params_libconfig coding rfsimulator dfts -- -j64" 
Log file for compilation is being written to: /home/gnb/OAIOSC/OAI_L1_2023w38/openairinterface5g/cmake_targets/log/all.txt
ERROR: 10 error. See /home/gnb/OAIOSC/OAI_L1_2023w38/openairinterface5g/cmake_targets/log/all.txt
compilation of oai_usrpdevif nr-softmodem nr-cuup params_libconfig coding rfsimulator dfts failed
build have failed
```

Excerpt of error message at the end of `all.txt`:

**[Case 1]** Error in: `nr-softmodem -> asn1_f1ap`
> The error that is sure to occur every time -i -c -C is re-executed [color=pink]
```bash=6000
...
[ 48%] Building C object openair2/F1AP/MESSAGES/CMakeFiles/asn1_f1ap.dir/xer_support.c.o
[ 48%] Linking C static library libasn1_f1ap.a
[ 48%] Built target asn1_f1ap
CMakeFiles/Makefile2:3302: recipe for target 'CMakeFiles/nr-softmodem.dir/rule' failed
make[1]: *** [CMakeFiles/nr-softmodem.dir/rule] Error 2
Makefile:1060: recipe for target 'nr-softmodem' failed
make: *** [nr-softmodem] Error 2
```

**[Case 2]** Error in: `nr-softmodem -> asn1_nr_rrc`
> The error that is sure to occur every time when re-executing **without** -i -c -C [color=pink]

```bash=5235
...
[ 96%] Built target asn1_nr_rrc
[ 96%] Built target asn1_lte_rrc
CMakeFiles/Makefile2:3302: recipe for target 'CMakeFiles/nr-softmodem.dir/rule' failed
make[1]: *** [CMakeFiles/nr-softmodem.dir/rule] Error 2
Makefile:1060: recipe for target 'nr-softmodem' failed
make: *** [nr-softmodem] Error 2
```

**[Case 3]** Error in: `nr-softmodem -> GTPV1U`
```bash=5000
...
[ 96%] Built target GTPV1U
CMakeFiles/Makefile2:3302: recipe for target 'CMakeFiles/nr-softmodem.dir/rule' failed
make[1]: *** [CMakeFiles/nr-softmodem.dir/rule] Error 2
Makefile:1060: recipe for target 'nr-softmodem' failed
make: *** [nr-softmodem] Error 2
```

# Additional

:::spoiler IP and port
![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/86eaf4ef-5582-4f0d-aac5-3402c4d3b0d4)

![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/ed809885-249e-4f54-9292-04e189d62764)

:::spoiler mcc
![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/f0e4d2b6-0ef6-4703-912d-8d774170569d)
:::

:::spoiler Q&A
-   Configuration Locations: 
    -   OAI: **Config file**
        -   CU :  OAI_CU/ci-scripts/conf_files/cu.band66.tm1.106PRB.usrpb210.conf
        -   PNF(L1) : OSC_L2/mwnl/oai_pnf_conf/oaiL1.nfapi.usrpb210.conf
    -   OSC: **Configure in the source code**
        -   OAIOSC/OSC_L2/src/du_app/du_cfg.c
        -   OAIOSC/OSC_L2/src/du_app/du_cfg.h

- Where the **LIBCONFIG** read from ?![image](https://github.com/NgKore47/Oai-CU-Osc-L2-Oai-L1/assets/81817735/2e7cebb9-2c29-46c3-8dc7-0dea501a4da5)

    - When we execute OAI L1, we include the location of the config file in the command, and **LIBCONFIG** will read the data from this config file.
        -  Path for include directive set to: ../../../../OSC_L2/mwnl/oai_pnf_conf
            -  OAIOSC/OAI_L1/common/config/libconfig/config_libconfig.c

---
with Robert

1. OAI Layer 1 reads from the USRP but doesn't transmit radio.

> Well, the logs says that the L1 does not receive enough samples from the USRP.
> 
> I don't know why, but the logs don't say whether you use USB3 (or 2, which is not good), so please check that.
> 
> You have errno 111, connection refused. The PNF cannot connect to OSC DU. The OSC DU says it accepted a connection from 127.0.0.1:33304 (line 906), what is this PNF?

2. It doesn't receive MIB and SIB from OSC DU, instead, it directly reads settings from the conf file.

> This has been solved in MR 2304, merged in tag 2023.w38. Please upgrade to that, or preferably newer.

---
11/24
I build like this, in nfapi-fixes 
```bash=
./build\_oai -c --ninja --nrUE --gNB 
```
you don't need USRP. 

If you still have problems, please attach file /home/gnb/OAIOSC/OAI\_L1\_2023w38/openairinterface5g/cmake\_targets/log/all.txt Robert



:::


:::spoiler Add timestamp in log
`ts` is a command-line utility in Linux that adds or converts timestamps to any output. It is part of the `moreutils` package and can be installed by running 
```shall=
sudo apt install moreutils
```

[The `ts` command can add timestamps to any output, including shell scripts and simple commands like `ping` and `traceroute` ](https://blog.csdn.net/m0_38059875/article/details/115401310)[1](https://blog.csdn.net/m0_38059875/article/details/115401310).

- stdbuf -oL 用於設置 ric_stub 的輸出緩衝模式為行緩衝模式，這樣可以確保輸出立即被傳送給管道。
- ts 用於為輸出添加時間戳

The `ts` command has several options that can be used to customize the timestamp format and behavior. [For example, the `-s` option starts the timestamp from the beginning of the program or operation, while the `-i` option starts the timestamp from the last timestamp ](https://blog.csdn.net/m0_38059875/article/details/115401310)[1](https://blog.csdn.net/m0_38059875/article/details/115401310). [The `-r` option converts existing timestamps in the input to relative timestamps, such as “15m5s ago” ](https://linux.die.net/man/1/ts)[2](https://linux.die.net/man/1/ts).

</details>
