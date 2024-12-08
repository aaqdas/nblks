# blk-switch: Rearchitecting Linux Storage Stack for μs Latency and High Throughput
blk-switch is a redesign of the Linux kernel storage stack that achieves μs-scale latencies while saturating 100Gbps links, even when tens of applications are colocated on a server. The key insight in blk-switch is that the multi-queue storage and network I/O architecture makes storage stacks conceptually similar to network switches. The main design contributions of blk-switch are:
- Multiple egress queues: enable isolating request types (prioritization)
- Request steering: avoids transient congestion (load valancing)
- Application steering: avoids persistent congestion (switch scheduling)

## 1. Overview
### Repository overview
- *block/* includes blk-switch core implementation. 
- *drivers/nvme/* includes remote storage stack kernel modules such as i10 and NVMe-over-TCP (nvme-tcp).
- *include/linux* includes small changes in some headers for blk-switch and i10.
- *osdi21_artifact/* includes scripts for OSDI21 artifact evaluation.
- *scripts/* includes scripts for getting started instructions.

### System overview
For simplicity, we assume that users have two physical servers (Host and Target) connected with each other over networks. Target server has actual storage devices (e.g., RAM block device, NVMe SSD, etc.), and Host server accesses the Target-side storage devices via the remote storage stack (e.g., i10, nvme-tcp) over the networks. Then Host server runs latency-sensitive applications (L-apps) and throughput-bound applications (T-apps) using standard I/O APIs (e.g., Linux AIO). Meanwhile, blk-switch plays a role for providing μs-latency and high throughput at the kernel block device layer.

### Getting Started Guide
Through the following three sections, we provide getting started instructions to install blk-switch and to run experiments.

   - **Build blk-switch Kernel (10 human-mins + 30 compute-mins + 5 reboot-mins):**  
blk-switch is currently implemented in the core part of Linux kernel storage stack (blk-mq at block device layer), so it requires kernel compilation and system reboot into the blk-switch kernel. This section covers how to build the blk-switch kernel and i10/nvme-tcp kernel modules. 
   - **Setup Remote Storage Devices (5 human-mins):**  
This section covers how to setup remote storage devices using i10/nvme-tcp kernel modules.
   - **Run Toy-experiments (5-10 compute-mins):**  
This section covers how to run experiments with the blk-switch kernel. 

The detailed instructions to reproduce all individual results presented in our OSDI21 paper is provided in the "[osdi21_artifact](https://github.com/resource-disaggregation/blk-switch/tree/master/osdi21_artifact)" directory.

## 2. Build blk-switch Kernel (with root)
blk-switch has been successfully tested on Ubuntu 16.04 LTS with kernel 5.4.43. Building the blk-switch kernel with NetChannel should be done on both Host and Target servers. It is assumed that both the servers are equipped with Mellanox Connect-X6 and MLNX OFED drivers are installed.  

1. Install the required packages using the command
    ```bash
    sudo apt-get install bc fio libncurses-dev gawk flex bison openssl libssl-dev dkms dwarves  \
                  libelf-dev libudev-dev libpci-dev libiberty-dev autoconf sysstat iperf
    ```

2. Install NVME Utility in Ubuntu 16.04
    ```bash
    cd ~ 
    wget http://security.ubuntu.com/ubuntu/pool/universe/n/nvme-cli/nvme-cli_1.9-1_amd64.deb
    sudo apt install ./nvme-cli_1.9-1_amd64.deb

    ```


**(Don't forget to be root)** 

3. Download Linux kernel source tree:
   ```
   sudo -s
   cd ~
   wget https://mirrors.edge.kernel.org/pub/linux/kernel/v5.x/linux-5.4.43.tar.gz
   tar xzvf linux-5.4.43.tar.gz
   ```

4. Download blk-switch source code and copy to the kernel source tree:

   ```
   git clone https://github.com/resource-disaggregation/blk-switch.git
   cd nblks
   cd ~/linux-5.4.43/
   git apply ../nblks/kernel.patch
   ```

4. Update kernel configuration:

   ```
   cp /boot/config-x.x.x .config
   make olddefconfig
   ```
   "x.x.x" is a kernel version. It can be your current kernel version or latest version your system has. Type "uname -r" to see your current kernel version.  
 
   Edit ".config" file to include your name in the kernel version.
   ```
   vi .config
   (in the file)
   ...
   CONFIG_LOCALVERSION="-jaehyun"
   ...
   ```
   Save the .config file and exit.   

5. Make sure i10 and nvme-tcp modules are included in the kernel configuration:

   ```
   make menuconfig

   - Device Drivers ---> NVME Support ---> <M> NVM Express over Fabrics TCP host driver
   - Device Drivers ---> NVME Support ---> <M>   NVMe over Fabrics TCP target support
   - Device Drivers ---> NVME Support ---> <M> i10: A New Remote Storage I/O Stack (host)
   - Device Drivers ---> NVME Support ---> <M> i10: A New Remote Storage I/O Stack (target)
   ```
   Press "Save" and "Exit"

6. Compile and install:

   ```
   (See NOTE below for '-j24')
   make -j16 bzImage
   make -j16 modules
   make modules_install
   make install
   ```
   NOTE: The number 16 means the number of threads created for compilation. Set it to be the total number of cores of your system to reduce the compilation time. Type "lscpu | grep 'CPU(s)'" to see the total number of cores:
   
   ```
   CPU(s):                16
   On-line CPU(s) list:   0-15
   ```

7. Edit "/etc/default/grub" to boot with your new kernel by default. For example:

   ```
   ...
   #GRUB_DEFAULT=0 
   GRUB_DEFAULT="1>Ubuntu, with Linux 5.4.43-ali"
   ...
   ```

8. Update the grub configuration and reboot into the new kernel.

   ```
   update-grub && reboot
   ```

9.  We need to define IPPROTO_VIRTUAL_SOCK for NetChannel applications. Add the two lines with sudo vi /usr/include/netinet/in.h (line 58):
```
   IPPROTO_VIRTUAL_SOCK = 19,      /* Virtual Socket.  */
#define IPPROTO_VIRTUAL_SOCK     IPPROTO_VIRTUAL_SOCK
```
10. Inside the NetChannel Repository. Run the following
```
cd ~/NetChannel
cd custom_socket/
sudo ./compile.sh

# Run your applications with 
sudo LD_PRELOAD=~/NetChannel/custom_socket/nd_socket.so ./program

```



11. Do the same steps 1--10 for both Host and Target servers.

12. When systems are rebooted, check the kernel version: Type "uname -r". It should be "5.4.43-(your name)".

13. Before we begin with the artifact, please edit `~/nblks/scripts/system_env.sh` to specify Target IP address, Network Interface name associated with the Target IP address, and number of cores.

   You can type "lscpu | grep 'CPU(s)'" to get the number of cores of your system.  


<!--
## 3. Setting Up Remote Storage Devices

### Target Configuration
**NOTE: Please edit `~/nblks/scripts/system_env.sh` to specify Target IP address, Network Interface name associated with the Target IP address, and number of cores before running `target_null.sh`.**  
   You can type "lscpu | grep 'CPU(s)'" to get the number of cores of your system.  

Use the script for a quick setup (for RAM null-blk devices):
   ```
   sudo -s
   cd ~/blk-switch/scripts/
   ./target_null.sh
   ```

### Host Configuration
**NOTE: please edit `~/blk-switch/scripts/system_env.sh` to specify Target IP address, Network Interface name associated with the Target IP address, and number of cores before running `host_tcp_null.sh`.**  
      You can type "lscpu | grep 'CPU(s)'" to get the number of cores of your system.  


1. Use the script for a quick setup:

      ```
      cd ~/blk-switch/scripts/
      (see NOTE below)
      ./host_tcp_null.sh
      ```
2. Check the remote storage device name you just created (e.g., `/dev/nvme0n1`):

   ```
   nvme list
   ```
-->
