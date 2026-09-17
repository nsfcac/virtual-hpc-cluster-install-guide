# Deploying a Virtual Cluster using VirtualBox (OpenHPC + SLURM)

Used to set up a small 3-node cluster of VMs for testing purposes. Adapted from OpenHPC's documentation.

Following the protocol provided in this guide:
https://github.com/openhpc/ohpc/releases/download/v3.4.GA/openhpc_3.4-install_guide-rocky9-warewulf-slurm-x86_64.pdf

## Preliminary Requirements - Cluster Setup & Network Configuration
- Make sure hardware virtualization is enabled in your system's BIOS.
- Download VirtualBox **and the VirtualBox Extension Pack**:
    - https://www.virtualbox.org/wiki/Downloads
    - The default installation settings will work.
    - To Install the Extension Pack into VirtualBox:
        - `File` -> `Tools` -> `Extensions`
        - Click on the `Install` button with the green plus at the top left of the pane.
        - Locate your downloaded pack in your filesystem and install.
            - (e.g. `Oracle_VirtualBox_Extension_Pack-7.2.14.vbox-extpack`)
## OpenHPC Master Node VM setup:
- Download the Rocky 9.6 Minimal ISO (`Rocky-9.6-x86_64-minimal.iso`) here:
    - https://dl.rockylinux.org/vault/rocky/9.6/isos/x86_64/
- Configure a new VM (`Machine` -> `New`):
    - VM Name and OS:
        - VM Name: `test-master` or something similar
        - ISO Image: `Rocky-9.6-x86_64-minimal.iso`
            - You'll need to find and select your ISO from your filesystem for the first boot.
        - Uncheck `Proceed with Unattended Installation` if it automatically selects the option.
        - OS: `Linux`
        - OS Distribution: `Red Hat`
        - OS Version: `Red Hat 9.x (64-bit)`
        - Click Next
    - Specify Virtual Hardware
        - Base Memory: `8192 MB`
        - Number of CPUs: `2`
        - Disk Size: `30.00 GB`
        - Click Next
    - Read Summary and Click Finish
- Configure the `test-master` VM's Network Interfaces
    - *NOTE: For MAC addresses, replace `XX:XX:XX:XX:XX:XX` with each respective address given by VirtualBox to the Network Interfaces.*
    - Right Click the new VM and select `Settings`.
    - Select the `Expert` option at the top of the window.
    - Select `Network` from the list.
    - `Enable` Adapter 1:
        - Attached to `NAT`
        - Adapter Type: `Intel PRO/1000 MT Desktop`
        - Promiscuous Mode: `Deny`
        - MAC Address: `XX:XX:XX:XX:XX`
        - Check the `Virtual Cable Connected` box.
    - `Enable` Adapter 2:
        - Attached to `Host-only Adapter`
        - Adapter Type: `Intel PRO/1000 MT Desktop`
        - Promiscuous Mode: `Deny`
        - MAC Address: `XX:XX:XX:XX:XX`
        - Check the `Virtual Cable Connected` box.
    - `Enable` Adapter 3:
        - Attached to `Internal Network`
        - Adapter Type: `Intel PRO/1000 MT Desktop`
        - Promiscuous Mode: `Deny`
        - MAC Address: `XX:XX:XX:XX:XX`
        - Check the `Virtual Cable Connected` box.
    - Click OK at the bottom of the Settings window to save your changes.
- Start up the VM and perform a regular installation of the OS.
    - Create an admin user and root. Save credentials someplace secure.
    - This guide will use `sudo...` commands instead of working through the `root` user.
- Login to `test-master` with your admin user account and update as needed:
    - `sudo dnf update`
    - Note for the future: `sudo dnf update --security` for only the security updates and none of the bugfixes or kernel updates.
-  The `test-master` VM will be the `OpenHPC Master Node`
    - Will be referred to as the "Master Node", "Master", or the Overall System Management Server (SMS) for this documentation.
    - Configure the network on the VM using `sudo nmtui` or a similar tool to confirm the following:
    - hostname: `master.openhpc.test`
    - [NAT] Internet-accessible Ethernet Interface: `enp0s3`
        - Static IP: `10.x.x.x`
        - MAC Address: `XX:XX:XX:XX:XX:XX`
    - [Host-Only] Data Center Network Ethernet Interface (& VM Connection): `enp0s8`
        - Static IP: `192.168.x.x`
        - MAC Address: `XX:XX:XX:XX:XX:XX`
    - [Internal] Cluster Backend Proivisioner and Manager Ethernet Interface: `enp0s9`
        - This will be configured in the upcoming steps! No worries if you do not see anything here yet.
        - Static IP: `10.0.0.1/24`
        - Gateway: `0.0.0.0`
        - MAC Address: `XX:XX:XX:XX:XX:XX`

## OpenHPC Compute Node VMs Setup
- Create two VMs **without an ISO** (don't select anything from the drop down menu), `test-compute-0` and `test-compute-1`, with 2 CPUS, 4GB of RAM, and 30GB of Disk. **Do not install anything on these VMs**.
    - The images will be generated and pushed out from the master node via PXE Booting.
- Set their boot order (Under Settings -> System -> Boot Order) to Network **only** - disable all other boot methods.
- Under their Network Settings (Settings -> Network -> Adapter 1,2,..), enable only one Network Adapter and attach it to the **Internal Network**.
- Note their respective MAC Addresses for later use in the installation steps. (`c_mac[0]` and `c_mac[1]`)  
- Under the Advanced tab in the Network Adapter, change their Adapter Type to `Intel PRO/1000 MT Server`.
    - The base iPXE booter VirtualBox uses does not support TFTP, which is required to provision the "bare metal" VMs with the compute images configured on the master node.


## OpenHPC Install Guide Inputs
Make sure to add the following list as environment variables to the SMS:  
ex.) `sms_name="master.openhpc.test"` -> `echo ${sms_name}` returns `master.openhpc.test`
| Input | Description |
|-|-|
|${sms_name} = `master.openhpc.test` | Hostname for the SMS server
|${sms_ip} = `10.0.0.1` | Internal IP address on the SMS server
|${sms_eth_internal} = `enp0s9` | Internal Ethernet interface on SMS
|${eth_provision} = `enp0s9` | Provisioning interface for computes
|${internal_netmask} = `255.255.255.0` | Subnet netmask for internal network
|${ntp_server} = `127.0.0.1` (on master node) | Local ntp server for time synchronization
|${num_computes} = `2` | Total # of desired compute nodes
|${c_ip[0]} =  `10.0.0.2` | Compute node 0 address
|${c_ip[1]} = `10.0.0.3` | Compute node 1 address
|${c_mac[0]} = `XX:XX:XX:XX:XX:XX` | MAC address for compute node 0 (enp0s9)
|${c_mac[1]} = `XX:XX:XX:XX:XX:XX` | MAC address for compute node 1 (enp0s9)
|${c_name[0]} = `c0` | Hostname for compute node 0
|${c_name[1]} = `c1` | Hostname for compute node 1
|${compute_regex} = `c*` | Regex matching all compute node names
|${compute_prefix} =`c` | Prefix for compute node names

## Installing OpenHPC Components
- The installation recipe assumes that the SMS host name is resolvable locally:
    - On the master node, do `sudo vi /etc/hosts` and ensure that the file is identical to the following lines and nothing more:
        ```
        10.0.0.1 master.openhpc.test master
        127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
        ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
        ```
- The installation recipe assumes that the local firewall running on the SMS host is disabled (make sure to turn it back on once everything has been configured correctly):
    - `sudo systemctl disable firewalld`
    - `sudo systemctl stop firewalld`
- Disable SELinux to allow the compute VMs to read from the master:
    - `sudo setenforce 0`
    - Edit the following line in `/etc/selinux/config`: 
        - `SELINUX=disabled`
- Edit the network configuration of the `enp0s9` interface:
    - `sudo nmtui`
    - Select the `Edit` option for the `enp0s9` network interface.
    - For the IPv4 configuration, select `Manual` instead of `Automatic`.
    - Set the Static IP: `10.0.0.1/24`
    - Set the Gateway: `0.0.0.0`
    - Save and close.
    - Restart the network: 
        - `sudo systemctl restart NetworkManager`
- Enable use of the OpenHPC repository by adding it to the local list of available package repositories:
    - `sudo dnf install http://repos.openhpc.community/OpenHPC/3/EL_9/x86_64/ohpc-release-3-1.el9.x86_64.rpm`
- Enable the CodeReady Builder (CRB) repository:
    - `sudo dnf install dnf-plugins-core`
    - `sudo dnf config-manager --set-enabled crb`
- Add provisioning services on the master node:
    - `sudo dnf install ohpc-base`
    - `sudo dnf install ohpc-warewulf`
    - `sudo dnf install hwloc-ohpc`
- HPC systems rely on synchronized clocks throughout the system and the NTP protocol can be used to facilitate this synchronization. Enable NTP services on the SMS host as root (I will use the master node's clock the synchronize. A separate server to handle this is often used instead):
    - `sudo systemctl enable chronyd.service`
    - `echo "local stratum 10" | sudo tee -a /etc/chrony.conf`
    - `echo "server ${ntp_server}" | sudo tee -a /etc/chrony.conf`
    - `echo "allow all" | sudo tee -a /etc/chrony.conf`
    - `sudo systemctl restart chronyd`
- Add resource management services on the master node.
    - Install slurm server meta-package:
        - `sudo dnf install ohpc-slurm-server`
    - Use ohpc-provided file for starting SLURM configuration:
        - `sudo cp /etc/slurm/slurm.conf.ohpc /etc/slurm/slurm.conf`
    - Setup default cgroups file:
        - `sudo cp /etc/slurm/cgroup.conf.example /etc/slurm/cgroup.conf`
    - Identify resource manager hostname on master host:
        - `sudo perl -pi -e "s/SlurmctldHost=\S+/SlurmctldHost=${sms_name}/" /etc/slurm/slurm.conf`
    - Edit the slurm configuration file to fit the context of this virtual cluster:
        - `sudo vi /etc/slurm/slurm.conf`
        - Edit the MpiDefault line to the following:
            - `MpiDefault=pmix`
        - Scroll down to the `COMPUTE NODES` section of the file. 
        - Edit the "NodeName" line as follows:
            - `NodeName=c[0-1] Sockets=1 CoresPerSocket=2 ThreadsPerCore=1 State=UNKNOWN`
        - Edit the "PartitionName" line as follows:
            - `PartitionName=normal Nodes=c[0-1] Default=YES MaxTime=24:00:00 State=UP`
        - Save and close.
- Infiniband and Omni-Path support services omitted for this deployment.
- Complete basic Warewulf setup for the master node:
    - Configure Warewulf provisioning to use desired internal interface
        - `sudo perl -pi -e "s/device = eth1/device = ${sms_eth_internal}/" /etc/warewulf/provision.conf`
    - Enable internal interface for provisioning (This should already be set up):
        - `sudo ip link set dev ${sms_eth_internal} up`
        - `sudo ip address add ${sms_ip}/${internal_netmask} broadcast + dev ${sms_eth_internal}`
    - Restart/enable relevant services to support provisioning
        - `sudo systemctl enable httpd.service`
        - `sudo systemctl restart httpd`
        - `sudo systemctl enable dhcpd.service`
        - `sudo systemctl enable tftp.socket`
        - `sudo systemctl start tftp.socket`
    
- Build initial Base Operating System (BOS) image. Begin by defining a directory structure on the master node that will represent the root filesystem of the compute node.
    - Define chroot location:
        - `export CHROOT=/opt/ohpc/admin/images/rocky9`
    - Build initial chroot image:
        - `sudo wwmkchroot -v rocky-9 $CHROOT`
    - Enable OpenHPC and EPEL repos inside chroot:
        - `sudo dnf --installroot $CHROOT install epel-release`
        - `sudo cp -p /etc/yum.repos.d/OpenHPC*.repo $CHROOT/etc/yum.repos.d`
    - Add additional components to include resource management client services, NTP support, and other additional packages to support the default OpenHPC environment.
    - Install compute node base meta-package:
        - `sudo dnf --installroot=$CHROOT install ohpc-base-compute`
    - Update the chroot environment with the master node's DNS configuration to access the remote repositories by hostname:
        - `sudo cp -p /etc/resolv.conf $CHROOT/etc/resolv.conf`
    - Next, include additional components to the compute instance:
        - Copy credential files into $CHROOT to ensure consistent uid/gids for slurm/munge at install. Note that these will be synchronized with future updates via the provisioning system.
            - `sudo cp /etc/passwd /etc/group $CHROOT/etc`
        - Add Slurm client support meta-package and enable munge and slurmd:
            - `sudo dnf --installroot=$CHROOT install ohpc-slurm-client`
            - `sudo chroot $CHROOT systemctl enable munge`
            - `sudo chroot $CHROOT systemctl enable slurmd`
        - Register Slurm server with computes (using "configless" option)
            - `echo SLURMD_OPTIONS="--conf-server ${sms_ip}" | sudo tee $CHROOT/etc/sysconfig/slurmd > /dev/null`
        - Add Network Time Protocol (NTP) support:
            - `sudo dnf --installroot=$CHROOT install chrony`
        - Identify master host as local NTP server:
            - `echo "server ${sms_ip} iburst" | sudo tee -a $CHROOT/etc/chrony.conf`  
        - Add kernel drivers (matching kernel version on SMS node):
            ```
            sudo dnf --installroot=$CHROOT install kernel-`uname -r`
            ```
        - Include modules user environment:
            - `sudo dnf --installroot=$CHROOT install lmod-ohpc`
- Customize system configuration:
    - Initialize warewulf database and ssh_keys:
        - `sudo wwinit database`
        - `sudo wwinit ssh_keys`
    - Add NFS client mounts of /home and /opt/ohpc/pub to base image:
        - `echo "${sms_ip}:/home /home nfs nfsvers=4,nodev,nosuid 0 0" | sudo tee -a $CHROOT/etc/fstab`
        - `echo "${sms_ip}:/opt/ohpc/pub /opt/ohpc/pub nfs nfsvers=4,nodev 0 0" | sudo tee -a $CHROOT/etc/fstab`
    - Export /home and OpenHPC public packages from master server:
        - `echo "/home *(rw,no_subtree_check,fsid=10,no_root_squash)" | sudo tee -a /etc/exports`
        - `echo "/opt/ohpc/pub *(ro,no_subtree_check,fsid=11)" | sudo tee -a /etc/exports`
        - Start the nfs-server service:
            - `sudo systemctl enable nfs-server`
            - `sudo systemctl start nfs-server`
        - `sudo exportfs -a`
        - `sudo systemctl restart nfs-server`
    - Optional customization steps skipped!
    - Import files:
        - Import local file-based credentials:
            ```
            sudo wwsh file import /etc/passwd
            sudo wwsh file import /etc/group
            sudo wwsh file import /etc/shadow
            ```
        - Import the cryptographic key that is required by the munge authentication library to be available on every host in the resource management pool:
            ```
            sudo wwsh file import /etc/munge/munge.key
            ```
        - Optional support for controlling IPoIB interfaces omitted.
- Finalizing provisioning configuration:
    - Assemble the bootstrap image.
        - Include drivers from kernel updates; needed if enable additional kernel modules on compute:
            ```
            export WW_CONF=/etc/warewulf/bootstrap.conf
            echo "drivers += updates/kernel/" | sudo tee -a $WW_CONF
            ```
        - Build bootstrap image:
            ``` 
            sudo wwbootstrap `uname -r`
            ```
    - Re-enable Firewall security for the master node
        - `sudo systemctl enable firewalld`
        - `sudo systemctl start firewalld`
        - Add SELinux enforcing policy to the compute nodes
            - `sudo dnf --installroot=$CHROOT install selinux-policy`
            - `sudo wwsh provision set --selinux=disabled c0`
            - `sudo wwsh provision set --selinux=disabled c1`
        - Add the internal network provisioning interface to the "internal" zone of the firewall:
            - `sudo firewall-cmd --zone=internal --add-interface=enp0s9 --permanent`
        - Allow DHCP and BOOTP for network IP allocation
            - `sudo firewall-cmd --zone=internal --add-service=dhcp --permanent`
        - Allow TFTP for PXE bootstrap/iPXE binaries
            - `sudo firewall-cmd --zone=internal --add-service=tftp --permanent`
        - Allow HTTP for Warewulf VNFS transfer processes
            - `sudo firewall-cmd --zone=internal --add-service=http --permanent`
        - Allow the NFS and RPC mountd/bind services for the compute nodes
            - `sudo firewall-cmd --zone=internal --add-service=nfs --permanent`
            - `sudo firewall-cmd --zone=internal --add-service=rpc-bind --permanent`
            - `sudo firewall-cmd --zone=internal --add-service=mountd --permanent`
        - Add SLURM Firewall configurations:
            - Add a locked port range at the bottom of `/etc/slurm/slurm.conf`:
                - `SrunPortRange=60001-63000`
            - Add the ports for the SLURM and munge services:
                - `sudo firewall-cmd --zone=internal --add-port=1110/tcp --permanent`
                - `sudo firewall-cmd --zone=internal --add-port=6817-6819/tcp --permanent`
                - `sudo firewall-cmd --zone=internal --add-port=60001-63000/tcp --permanent`
        - Reload the firewall to enforce the above changes
            - `sudo firewall-cmd --reload`
    - Assemble the Virtual Node File System (VNFS) image:
        - Note: you might need to increase the `max_allowed_packet` variable, controlling the maximum number of bytes allowed per package for the local MySQL/MariaDB database:
            - `sudo vi /etc/my.cnf.d/mariadb-server.cnf`
            - Find or add the `[mariadb-10.5]` block and add the following:
                - `max_allowed_packet = 512M`
            - `sudo systemctl restart mariadb`
            - Edit the Warewulf database to override its max chunk size:
                - `sudo vi /etc/warewulf/database.conf`
                - Find the line `database chunk size...` and uncomment it.
        - Assemble a VNFS capsule from the chroot environment defined for the compute instance:
            - `sudo wwvnfs --chroot $CHROOT`
    - Register nodes for provisioning
        - Set provisioning interface as the default networking device:
            ```
            echo "GATEWAYDEV=${eth_provision}" | sudo tee /tmp/network.$$
            sudo wwsh -y file import /tmp/network.$$ --name network
            sudo wwsh -y file set network --path /etc/sysconfig/network --mode=0644 --uid=0
            ```
        - Add nodes to Warewulf data store: 
            ```
            for ((i=0; i<$num_computes; i++)) ; do
                sudo wwsh -y node new ${c_name[i]} --ipaddr=${c_ip[i]} --hwaddr=${c_mac[i]} -D ${eth_provision}
            done
            ```
        - Additional step required if desiring to use predictable network interface naming schemes (e.g. enp0s9) instead of eth*:
            ```
            export kargs="${kargs} net.ifnames=1,biosdevname=1"
            sudo wwsh provision set --postnetdown=1 "${compute_regex}"
            ```
        - Define provisioning image for hosts:
            ```
            sudo wwsh -y provision set "${compute_regex}" --vnfs=rocky9 --bootstrap=`uname -r` --files=dynamic_hosts,passwd,group,shadow,munge.key,network
            ```
        - Omit defining IPoIB network settings (not planning to mount Lustre/BeeGFS over IB)
    - Restart dhcp / update PXE:
        - `sudo systemctl restart dhcpd`
        - `sudo wwsh pxe update`
    - Skipped optional kernel arguments.
    - Skipped stateful provisioning.

- Boot compute nodes:
    - At this point, the master node VM should be able to boot the newly defined compute nodes! Go into VirtualBox, and power up the `compute-0` and `compute-1` VMs **IN THAT ORDER**. They should automatically latch onto the master node's compute image exports and download the appropriate files. (Note that for bringing up the cluster again, i.e., restarting the master and compute nodes, you need to "power cycle" the VMs (turning them completely off - "Power off the machine" option) instead of using the "restart" option in Virtual Box)
    - Once both compute nodes have been fully booted up (you can see the login prompt on both compute nodes), verify that the compute hosts are available via ssh:
        - `sudo pdsh -w c[0-1] uptime`
        - Output on the master node:
            ```
            c0:  09:22:29 up 26 min,  0 users,  load average: 0.00, 0.00, 0.00
            c1:  09:22:30 up 24 min,  0 users,  load average: 0.00, 0.00, 0.00
            ```
- Install OpenHPC Development Components
    - Install development tools:
        ```
        sudo dnf install ohpc-autotools
        sudo dnf install EasyBuild-ohpc
        sudo dnf install hwloc-ohpc
        sudo dnf install spack-ohpc
        sudo dnf install valgrind-ohpc
        ```
    - Install compilers:
        ```
        sudo dnf install gnu15-compilers-ohpc
        ```
    - Install MPI stacks:
        - `sudo dnf install openmpi5-pmix-gnu15-ohpc mpich-ofi-gnu15-ohpc`
        - Omitted installation for `MVAPICH2` and `MVAPICH2 (psm2)`.
    - Install Performance Analysis tools:
        - `sudo dnf install ohpc-gnu15-perf-tools`
    - Setup default development environment
        - Set up a default development environment for parallel programs requiring MPI so that compilation is more straightforward.
            - `sudo dnf install lmod-defaults-gnu15-openmpi5-ohpc`
    - Install available 3rd party libraries and tools:
        ```
        sudo dnf install ohpc-gnu15-serial-libs
        sudo dnf install ohpc-gnu15-io-libs
        sudo dnf install ohpc-gnu15-python-libs
        sudo dnf install ohpc-gnu15-runtimes
        sudo dnf install ohpc-gnu15-mpich-parallel-libs
        sudo dnf install ohpc-gnu15-openmpi5-parallel-libs
        ```
    - Omitted the installation of the optional development tool builds.
- Once the above steps have been completed, you should now have a functional virtual 3-node OpenHPC cluster running in VirtualBox. The next step is to deploy Slurm! Take a snapshot of the master node after completing the above steps and move on to the next section.

## Deploying SLURM

Starting at Section 5 (Page 31 in the linked PDF)

- Resource Manager Startup
    - The following commands can be used to startup the necessary services to support resource management under Slurm:
        - Start munge and slurm controller on the master node:
            - `sudo systemctl enable munge`
            - `sudo systemctl enable slurmctld`
            - `sudo systemctl start munge`
            - `sudo systemctl start slurmctld`
        - Start slurm clients on the compute nodes:
        - **NOTE: You will need to start these services each time you spin up the cluster!**
            - `sudo pdsh -w ${compute_prefix}[0-$((${num_computes} - 1))] systemctl start munge`
            - `sudo pdsh -w ${compute_prefix}[0-$((${num_computes} - 1))] systemctl start slurmd`
        - Notes for later:
            - Full slurm startup takes about 2-3 min. for each node. Be patient!
            - Check the status of the compute nodes:
                - `sinfo`
            - Change the state of the compute nodes to run jobs if needed:
                - `sudo scontrol update nodename=c0 state=idle`
                - `sudo scontrol update nodename=c1 state=idle`
- Post-boot compute node configuration
    - Omitted this section since we did not install NHC on the server.
- Run a Test Job
    - Add a test user on the master node that can be used to run an example job:
        - `sudo useradd -m test`
    - Update the previously registered credential files in the Warewulf database to propagate the addtion of the new test user:
        - `sudo wwsh file resync passwd shadow group`
        - Note: After re-syncing to notify Warewulf of file modifications made on the master host, it should take approximately 5 minutes for the changes to propagate. However, you can also manually pull the changes from compute nodes via the following:
            - `sudo pdsh -w ${compute_prefix}[1-${num_computes}] /warewulf/bin/wwgetfiles`
    - Interactive Execution
        - Switch to "test" user:
            - `sudo su - test`
        - Compile MPI "hello world" example:
            - `mpicc -O3 /opt/ohpc/pub/examples/mpi/hello.c`
        - Submit interactive job request and use prun to launch executable:
            - `salloc -n 2 -N 2`
            - `prun ./a.out`
            - Output:
                ```
                [prun] Master compute host = c0
                [prun] Resource manager = slurm
                [prun] Launch cmd = srun --mpi=pmix -- ./a.out  (family=openmpi5)
                
                 Hello, world (2 procs total)
                    --> Process #   0 of   2 is alive. -> c0
                    --> Process #   1 of   2 is alive. -> c1
                ```
    - Batch Execution
        - We will use the same compiled executable from the previous section.
        - Copy the example job script:
            - `cp /opt/ohpc/pub/examples/slurm/job.mpi .`
        - Examine contents (and edit to set desired job sizing characteristics)
            - `cat job.mpi`
            - Output:
                ```
                #!/bin/bash

                #SBATCH -J test               # Job name
                #SBATCH -o job.%j.out         # Name of stdout output file (%j expands to jobId)
                #SBATCH -N 2                  # Total number of nodes requested
                #SBATCH -n 2                 # Total number of mpi tasks requested
                #SBATCH -t 01:30:00           # Run time (hh:mm:ss) - 1.5 hours

                # Launch MPI-based executable

                prun ./a.out

                ```
        - Submit job for batch execution:
            - `sbatch job.mpi`
        - Check output file (the submitted job had an ID of 3):
            ```
            [prun] Master compute host = c0
            [prun] Resource manager = slurm
            [prun] Launch cmd = srun --mpi=pmix -- ./a.out  (family=openmpi5)
                --> Process #   1 of   2 is alive. -> c1
            
             Hello, world (2 procs total)
                --> Process #   0 of   2 is alive. -> c0
            ```
- Following these tests, we can confirm that we have a successfully deployed SLURM on the OpenHPC cluster. Take a snapshot of the master node and relax for a bit!
