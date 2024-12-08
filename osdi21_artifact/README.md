# USENIX OSDI 2021 Artifact Evaluation

## 1. Detailed Instructions
Now we provide how to use our scripts to reproduce the results in the paper. 

### Run configuration scripts (with root)
You should be root from now on. If you already ran some configuration scripts below while doing the getting started instruction, you **SHOULD NOT** run those scripts -- `target_null.sh`, `host_tcp_null.sh`, and `host_i10_null.sh`.

**(Don't forget to be root)**

1. At Both Target and Host:  
 Please edit `~/nblks/scripts/system_env.sh` to specify Target IP address, Network interface name associated with the Target IP address, and number of cores of your system. You can type "lscpu | grep 'CPU(s)'" to get the number of cores of your system.   
 Then run `system_setup.sh`:
   ```
   sudo -s
   cd ~/blk-switch/scripts/
   ./system_setup.sh
    ```
   **NOTE:** `system_setup.sh` enables aRFS on Mellanox ConnextX-5 NICs. For different NICs, you may need to follow a different procedure to enable aRFS (please refer to the NIC documentation). If the NIC does not support aRFS, the results that you observe could be significantly different. (We have not experimented with setups where aRFS is disabled).
   
   The below error messages from `system_setup.sh` is normal. Please ignore them.
   ```
   Cannot get device udp-fragmentation-offload settings: Operation not supported
   Cannot get device udp-fragmentation-offload settings: Operation not supported
   ```

2. At Target:  
   ```
   sudo -s
   cd ~/blk-switch/scripts/
   ./target_null.sh
   ```   
   If you ran `target_null.sh` twice by mistake and got several errors like "Permission denied", please reboot the both servers and restart from "Run configuration scripts".
   
   
3. At Host:  
 After running the scripts below, you will see that 2-4 remote storage devices are created (type `nvme list`).

   ```
   sudo -s
   cd ~/blk-switch/scripts/
   ./host_tcp_null.sh
   ./host_i10_null.sh
   ```

### Linux and blk-switch Evaluation (with root)
Now you will run evaluation scripts at Host server. We need to identify newly added remote devices to use the right one for each script.  

We assume your Host server now has 2 remote devices that have different '*Namespace number*' as follows:
- `/dev/nvme0n1`: null-blk device for blk-switch (*Namespace: 10*)
- `/dev/nvme1n1`: null-blk device for Linux (*Namespace: 20*)


Type `nvme list` and check if the device names are matching the Namespace above. If this is your case, then you are safe to go. The default configurations in our scripts are:  

Before the script please specify the cores in `nr_lapp.pl` Increasing L-app load (6 mins):

   ```
   cd ~/blk-switch/osdi21_artifact/blk-switch/
   ./linux_fig7.pl
   ./blk-switch_fig7.pl
   ```

