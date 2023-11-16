# Virtualization Tutorial

-----

Hello! These are some notes for the UTM version ( which took me *sometime* to setup following the Virtualbox tutorial :( ), feel free to ask for help or write me for corrections, hopefully this goes smooth! I'm following the template from the Virtualbox setup and will highlight only the major differences in setup  

good luck...  
Adriano D.

-----

In this tutorial, we will learn how to build a cluster of Linux machines on our local environment using Virtualbox. 
Each machine will have two NICs one internal and one to connect to WAN .
We will connect to them from our host windows machine via SSH.

This configuration is useful to try out a clustered application which requires multiple Linux machines like kubernetes or an HPC cluster on your local environment.
The primary goal will be to test the guest virtual machine performances using standard benckmaks as HPL, STREAM or iozone and compare with the host performances.

Then we will install a slurm based cluster to test parallel applications

## GOALs
In this tutorial, we are going to create a cluster of four Linux virtual machines.

* Each machine is capable of connecting to the internet and able to connect with each other privately as well as can be reached from the host machine.
* Our machines will be named cluster01, cluster02, ..., cluster0X.
* The first machine: cluster01 will act as a master node and will have 1vCPUs, 2GB of RAM and 25 GB hard disk.
* The other machines will act as worker nodes will have 1vCPUs, 1GB of RAM and 10 GB hard disk.
* We will assign our machines static IP address in the internal network: 192.168.0.1, 192.168.0.22, 192.168.0.23, .... 192.168.0.XX.

## Prerequisite

* UTM installed in your linux/windows/Apple (UTM in Arm based Mac)
* ubuntu server (I used ARM version) 22.04 LTS image to install
* SSH client to connect

## Create virtual machines on Virtualbox
We create one template that we will use then to deply the cluster and to make some performance tests and comparisons

(For the initial setup you can use more resources and then resize the template later so that the machine is more powerful and installation is faster)

Create the template virtual machine which we will named "template" with (at least) 1vCPUs, 1GB of RAM and 25 GB hard disk. 

You can use Ubuntu 22.04 LTS server (https://ubuntu.com/download/server)

### Network setup:
Right click on your machine -> edit -> in Devices be sure there's a network (otherwise add it).
Configure it like this

<img width="678" alt="Screenshot 2023-11-11 at 17 55 58" src="https://github.com/JunkyByte/Cloud-Basic-2023/assets/24314647/fc1f2ccf-b69a-41eb-9b18-878b3f6f2ad9">

### Boot and install ubuntu

The VM is now able to access the network to download the software and updates for the LTS. 

When you are prompted for the "Guided storage configuration" panel keep the default installation method: use an entire disk. 

When you are prompted for the Profile setup, you will be requested to define a server name (template) and super user (e.g. user01) and his administrative password.

*Also, enable open ssh server in the software selection prompt.*

Finish the installation and then shutdown the VM.

Remove the CD so that you are sure is booting correctly

<img width="595" alt="image" src="https://github.com/JunkyByte/Cloud-Basic-2023/assets/24314647/cc9983bd-68ba-4d32-ab82-6ba099d86dd1">

Boot again and check if internet works
```
ping google.com
```

update the software:

```
$ sudo apt update
...

$ sudo apt upgrade
```

You can check the DHCP-assigned IP address by entering the following command:

```shell
hostname -I
```

You will get an output similar to this:
```
10.0.2.15
```

This is the default IP address assigned by your network DHCP. Note that this IP address is dynamic and can change or worst still, get assigned to another machine.

Now install some useful additional packages:

```
$ sudo apt install net-tools
```

### Login/master node

You could create templates from here but I think you have to do a lot more work later so I will do a few more steps and create a template only when I'm almost done.

I want to setup the static ip for this VM now but working from the machine is terrible, I like to ssh so let's setup that:

### Port forwarding on login/master node

Turn off the VM  
Right click on it -> edit -> New (Network) and setup it like this  

<img width="656" alt="image" src="https://github.com/JunkyByte/Cloud-Basic-2023/assets/24314647/b466bbc5-f2e6-48db-9ec4-0055a5e8b031">

Now on the left side of the menu you will see the Port forwarding (under this network), click on it and setup it as follows (you just need to insert the ports! Do not change the ips)

<img width="662" alt="image" src="https://github.com/JunkyByte/Cloud-Basic-2023/assets/24314647/11d89b53-8b23-41c0-9465-868f5d31e6ab">

The second port (the host port) will be different on each other machine we will create, so remember that when you clone a new one you must come here and edit it as you cannot have the same on multiple machines. I suggest you to do incremenetally (22, 23 ...) with 22 being master and 23 ... the computing nodes we will create later.  
(If 22 does not work just change it like to something like 2222, 2223 ...)

Launch the VM  
If everything is correct you should be able to ssh to it with

```
ssh -p 22 your_vm_username@localhost
```

Does it work?  

You will have to enter the password at each login, let's change way using keys  
If you want a passwordless access you need to generate a ssh key or use an ssh key if you already have it.

If you don’t have public/private key pair already, run ssh-keygen and agree to all defaults. 
This will create id_rsa (private key) and id_rsa.pub (public key) in ~/.ssh directory.  
Check that that is the actual key created if you have already one in your system path might differ.

Copy host public key to your VM:

```
scp -P 22 ~/.ssh/id_rsa.pub your_vm_username@localhost:~
```

Connect to the VM and add host public key to ~/.ssh/authorized_keys:

```
ssh -p 22 your_vm_username@localhost
mkdir ~/.ssh
chmod 700 ~/.ssh
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 644 ~/.ssh/authorized_keys
exit
```

Now you should be able to ssh to the VM without password.  
If it does not work and keeps asking your password do some googling (for instance disable password authentication on `sshd_config`)

If your ssh works correctly and you do not want to use graphical interface anymore for the VM (which is the cool pro move) go Edit -> Select the Display device in devices and remove it.
Now when you boot the VM there's no graphical interface at all and you can only connect to it through ssh. (Now you are a pro)

### Static IPs

Let's go back setting a static IP for the VM.

First find interface name (might be same as me `enp0s2`)

```
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: enp0s1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether a6:96:c2:54:8c:00 brd ff:ff:ff:ff:ff:ff
3: enp0s2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether c6:20:e4:d9:31:bd brd ff:ff:ff:ff:ff:ff
```

Now we configure the adapter. To do this we will edit the netplan file:  
(Use nano if you are not a vim fan)

```
$ sudo vim /etc/netplan/00-installer-config.yaml

# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s1:
      dhcp4: true
    enp0s2:
      dhcp4: no
      addresses: [192.168.64.0/24]
  version: 2
```

Using `192.168.0.1` (as the professor does) *DID NOT* work at all to me.  
I opted to use `192.168.64.0` for master node and then `192.168.64.[1,2,...]` for the compute nodes.

save and apply the configuration

```
$ sudo netplan apply
```

Change the hostname:
```
$ sudo vim /etc/hostname

cluster01
```

Now reboot the VM and check if the static IP is correctly assigned!
```
$ hostname -I
10.0.2.15 192.168.64.0 fec0::a496:c2ff:fe54:8c00 fd30:2e54:53c8:27c5:c420:e4ff:fed9:31bd
```

If it does good job!
Now let's setup the hosts file.

Edit the hosts file to assign names to the cluster that should include names for each node as follows:

```
$ sudo vim /etc/hosts

127.0.0.1 localhost
192.168.64.0 cluster01

192.168.64.1 cluster02
192.168.64.2 cluster03
192.168.64.3 cluster04
192.168.64.4 cluster05
192.168.64.5 cluster06

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Then we install a DNSMASQ server to dynamically assign the IP and hostname to the other nodes on the internal interface and create a cluster [1].

```
$ sudo systemctl disable systemd-resolved
Removed /etc/systemd/system/multi-user.target.wants/systemd-resolved.service.
Removed /etc/systemd/system/dbus-org.freedesktop.resolve1.service.
$ sudo systemctl stop systemd-resolved
```

Then 

```
$ ls -lh /etc/resolv.conf
lrwxrwxrwx 1 root root 39 Jul 26 2018 /etc/resolv.conf ../run/systemd/resolve/stub-resolv.conf
$ sudo unlink /etc/resolv.conf

```

Create a new resolv.conf file like this

```
$ echo nameserver 192.168.64.0 | sudo tee /etc/resolv.conf
```

Install dnsmasq

```
$ sudo apt install dnsmasq -y
```

To find and configuration file for Dnsmasq, navigate to /etc/dnsmasq.conf. Edit the file by modifying it with your desired configs. Below is minimal configurations for it to run and support minimum operations.

```
$ vim /etc/dnsmasq.conf

port=53
bogus-priv
strict-order
expand-hosts
dhcp-range=192.168.64.1,192.168.64.8,255.255.255.0,12h
server=8.8.8.8
# dhcp-option=option:dns-server,192.168.64.0
dhcp-option=3

```

When done with editing the file, close it and restart Dnsmasq to apply the changes. 
```
$ sudo systemctl restart dnsmasq
```

Check if it is working

```
$ host cluster01
cluster01 has address 192.168.64.0
```

If it does, good job!  
(This part took me probably 2 hours alone and might be wrong, we are not using google DNS but it works, whatever).

Now shutdown the machine and do a clone (I suggest you do 2 and one you call it "template common" just to be sure if you make mistakes later), now we will have differences in setup for master and compute nodes.  
You should have 3 machines, the one you worked up to now called cluster01 and two others, one is a "template common" and the last one will be used for compute nodes.  

Let's continue a bit with cluster01 setup, turn it on again...  
To build a cluster we also need a distributed filesystem accessible from all nodes. 
We use NFS.

```
$ sudo apt install nfs-kernel-server
$ sudo mkdir /shared
$ sudo chown -R your_vm_username /shared
$ sudo chmod 777 /shared
```

Modify the NFS config file:

```
$ sudo vim /etc/exports

/shared 192.168.64.0/24(rw,sync,root_squash,subtree_check)
```

Restart the server

```
$ sudo systemctl enable nfs-kernel-server
$ sudo systemctl restart nfs-kernel-server
```

Shut it down again. Now let's setup the computing nodes for a bit.  
Just to be sure assign different MAC addresses to the network devices of your new machines everytime you clone them (in this case apply this to the computing node we are about to start), to do so Edit -> On each network device -> MAC Address -> Random

Let's setup your first computing node. Remember I told you you have to change the port for ssh connection otherwise you won't be able to connect to it while cluster01 is on. To do so on the compute node -> Edit -> Port Forwarding and update the port (I explained above the meaning).

Now you can connect to this machine using `ssh -p 23 your_vm_username@localhost` (notice is 23 or whatever port you put)

### Computing nodes

*This must be done every time you will create a new computing node (after you create a template for computing nodes you will just have to change the ip related stuff and update the hostname in both hostname and hosts files)*

Setup the netplan file with new static ip for this machine (this is first compute node so I will use `192.168.64.1`...

```
$ sudo vim /etc/netplan/00-installer-config.yaml

# This is the network config written by 'subiquity'
network:
  ethernets:
    enp0s1:
      dhcp4: true
    enp0s2:
      dhcp4: false
      addresses: [192.168.64.1/24]
  version: 2
```

and apply the configuration

```
$ sudo netplan apply
```

Change the hostname to a new index to remember each machine:
```
$ sudo vim /etc/hostname

cluster02
```

Set the proper dns server (assigned with dhcp):

```
$ sudo rm /etc/resolv.conf
```

then

```
$ sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

Setup hosts file to a better version for each computing node

```
$ sudo vim /etc/hosts

127.0.0.1 localhost
127.0.1.1 cluster02

192.168.64.0 cluster01

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Reboot the machine.

At reboot you will see that the machine will have a the static ip address (does it?)

```
$ hostname -I
10.0.2.15 192.168.64.1 fec0::fc5a:9fff:fee3:77e fd30:2e54:53c8:27c5:d058:bfff:fe9f:d05
```

Install dnsmasq (trick necessary to install the cluster later)
```
$ sudo apt install dnsmasq -y
$ sudo systemctl disable dnsmasq
```

Now you should we able to connect *from cluster01 to cluster02 (both must be on).*

```
$ ssh your_vm_username@cluster02
your_vm_username@cluster02:~$
(Be careful to be on the right machine to issue future commands)
(As you have same username you can also just do ssh cluster02)
```

To access the new machine without password you can proceed like described above.  

*The following must be done on cluster01*

Run ssh-keygen and agree to all defaults. 
This will create id_rsa (private key) and id_rsa.pub (public key) in ~/.ssh directory.

Copy host public key to your VM:

```
scp  ~/.ssh/id_rsa.pub your_vm_username@cluster02:~
```

Connect to the VM and add host public key to ~/.ssh/authorized_keys:

```
ssh your_vm_username@cluster02
mkdir ~/.ssh
chmod 700 ~/.ssh
cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 644 ~/.ssh/authorized_keys
exit
```

* Done, now you should be able to connect from cluster01 to cluster02 without password! *  
(Test it)

Now back to compute node configuration...  
Configure the shared filesystem

```
$ sudo apt install nfs-common
$ sudo mkdir /shared
$ sudo chmod 777 /shared
```

Mount the shared directory and test it

```
$ sudo mount -t nfs 192.168.64.0:/shared /shared
$ touch /shared/pippo
```
If everything will be ok you will see the "pippo" file in all the nodes.

To authomatically mount at boot edit the /etc/fstab file:

```
$ sudo vim /etc/fstab

```
 
Append the following line at the end of the file

```
192.168.64.0:/shared /shared nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0
``` 

## Install a SLURM based cluster
Here I will describe a simple configuration of the slurm management tool for launching jobs in a really simplistic Virtual cluster. I will assume the following configuration: a main node (cluster01) and 3 compute nodes (cluster03 ... VMs). I also assume there is ping access between the nodes and some sort of mechanism for you to know the IP of each node at all times (most basic should be a local NAT with static IPs)

^^^ Do we have a ping system working?  
On cluster 01 try to run `ping cluster02` it should work! :)

Slurm management tool work on a set of nodes, one of which is considered the master node, and has the slurmctld daemon running; all other compute nodes have the slurmd daemon. 

All communications are authenticated via the munge service and all nodes need to share the same authentication key. Slurm by default holds a journal of activities in a directory configured in the slurm.conf file, however a Database management system can be set. All in all what we will try to do is:

 * Install munge in all nodes and configure the same authentication key in each of them
 * Install gcc, openmpi and configure them
 * Configure the slurmctld service in the master node
 * Configure the slurmd service in the compute nodes
 * Create a basic file structure for storing jobs and jobs result that is equal in all the nodes of the cluster
 * Manipulate the state of the nodes, and learn to resume them if they are down
 * Run some simple jobs as test
 * Set up MPI task on the cluster

* On both machines (cluster01 and cluster02 follow these instructions) *

### Install gcc and OpenMPI

```
$ sudo apt install gcc-12 openmpi-bin openmpi-common
```
Configure gcc-12 as default:
```
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-12 100
```
Test the installation:

```
$ gcc --version 
gcc (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
and MPI

```
$ mpicc --version
gcc (Ubuntu 12.3.0-1ubuntu1~22.04) 12.3.0
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

### Install MUNGE

Lets start installing munge authentication tool using the system package manager:

```
$ sudo apt-get install -y libmunge-dev libmunge2 munge
```

munge requires that we generate a key file for testing authentication, for this we use the dd utility, with the fast pseudo-random device /dev/urandom.  
* cluster01 node do: *  
```
$ sudo dd if=/dev/urandom bs=1 count=1024 | sudo tee /etc/munge/munge.key > /dev/null
$ sudo chown munge:munge /etc/munge/munge.key
$ sudo chown -R munge:munge /var/lib/munge
$ sudo chown munge:munge /var/log/munge
$ sudo chown munge:munge /etc/munge
$ sudo chown munge:munge /var/log/munge/munged.log
$ sudo chmod 400 /etc/munge/munge.key
$ sudo chmod 700 /etc/munge
```

Copy the key on cluster02 with scp (if `/etc/munge/` is too protected you won't be able to scp directly, scp it to `/home/your_vm_username` first and the from cluster02 use `sudo mv` to move the key to the correct folder).  

* On cluster02 *

I had to also be sure UID and GID are the same for munge user in different machines, to do so in cluster01 run
```
$ grep munge /etc/passwd /etc/group
/etc/passwd:munge:x:117:121::/nonexistent:/usr/sbin/nologin
/etc/group:munge:x:121:
```
The first number `117` here is the UID while `121` here is GID.  
We want same UID and GID for munge also on cluster02.  
On cluster 02 run:
```
sudo vim /etc/passwd
sudo vim /etc/groupd
```
And change the numbers associated with munge to your numbers (`117` and `121` as above).  

Now that the key is in correct path run all commands above (the ones about `chown` and `chmod` of munge directories, *do not run the `dd one`*)

I would reboot all machines.

Now you can test communication, on both machines try to run the following two commands, you should see success statuses.

```
$ munge -n | unmunge
$ munge -n | ssh cluster03 unmunge
```

Trouble? This part was terrible for me. If you encounter problems with munge you can check if munge encountered errors with `journalctl -xe | grep munge` and the service status with `sudo systemctl status munge`, try to fix the errors it complains about.

### Install Slurm

```
$ sudo apt-get install -y slurmd
```

Copy the slurm configuration file of GIT repository (the one in UTM folder) to '/etc/slurm/slurm.conf' directory of in cluster01 and cluster02.

On cluster01

```
$ sudo apt install -y slurmctld
$ sudo systemctl enable slurmctld
$ sudo systemctl start slurmctld

$ sudo systemctl enable slurmd
$ sudo systemctl start slurmd
```

On cluster02

```
$ sudo systemctl enable slurmd
$ sudo systemctl start slurmd
```

Now you can test the environment:

```
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      2   idle cluster[02-03]
```

(This is my correct state but also cluster03 does not exist yet in your setup)
Probably it will not work or report some errors in the STATE :)

On cluster02 run:
```
$ sinfo -N -r  -l
NODELIST   NODES PARTITION       STATE CPUS    S:C:T MEMORY TMP_DISK WEIGHT AVAIL_FE REASON
cluster02      1    debug*        idle 1       1:1:1   1464        0      1   (null) none
cluster03      1    debug*        idle 1       1:1:1   1464        0      1   (null) none
```

You see there's a certain amount of memory and number of cpus! We must reflect them in the `slurm.conf`. On both machine do

```
sudo vim /etc/slurm/slurm.conf

(Find the lines with nodes description) and update Real memory and CPUs with correct numbers:
NodeName=cluster02 NodeAddr=192.168.64.1 CPUs=1 RealMemory=1464
```

Now the setup should be okay but still it might not work, let's reset the state of our machines. 
From cluster01 run

```
sudo scontrol reconfigure cluster02
sudo scontrol update nodename=cluster02 state=resume
```

If it does not work try to also reboot both machines and rerun ^ commands on cluster01

Test a job:

```
$ srun hostname
cluster02
```

Yay! Shut down cluster02. Create a clone of it and call it templatecomputenode, now create another clone for cluster03 :) As above change the hosts, hostname and netplan to another ip.  

Turn all machines on. Now running sinfo should give you

```
$ sinfo
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      2   idle cluster[02-03]
```

the rest of the tutorial has not been checked and is from the original tutorial of the professor.

Have fun :)

## Testing VMM enviroment

### Test VM performance
We propose 3 different tests to study VMs performances and VMM capabilities.

* Test the network isolation cloning a new VM from template then change the name of the "Internal Netwok" to "testnetwork", then try to access to the new VM from the cluster
* Using HPCC   compare the host and the guest performance in terms of CPU flops and memory bandwidth
* Using iozone compare host IO performance with guest IO performance

For windows based system it may be necessary to install Linux subsystem.


Install and use hpcc:

```
sudo apt install hpcc
```

HPCC is a suite of benchmarks that measure performance of processor,
memory subsytem, and the interconnect. For details refer to the
HPC~Challenge web site (\url{http://icl.cs.utk.edu/hpcc/}.)

In essence, HPC Challenge consists of a number of tests each
of which measures performance of a different aspect of the system: HPL, STREAM, DGEMM, PTRANS, RandomAccess, FFT.

If you are familiar with the High Performance Linpack (HPL) benchmark
code (see the HPL web site: http://www.netlib.org/benchmark/hpl/}) then you can reuse the input
file that you already have for HPL. 
See http://www.netlib.org/benchmark/hpl/tuning.html for a description of this file and its parameters.
You can use the following sites for finding the appropriate values:

 * Tweak HPL parameters: https://www.advancedclustering.com/act_kb/tune-hpl-dat-file/
 * HPL Calculator: https://hpl-calculator.sourceforge.net/

The main parameters to play with for optimizing the HPL runs are:

 * NB: depends on the CPU architecture, use the recommended blocking sizes (NB in HPL.dat) listed after loading the toolchain/intel module under $EBROOTIMKL/compilers_and_libraries/linux/mkl/benchmarks/mp_linpack/readme.txt, i.e
   * NB=192 for the broadwell processors available on iris
   * NB=384 on the skylake processors available on iris
 * P and Q, knowing that the product P x Q SHOULD typically be equal to the number of MPI processes.
 * Of course N the problem size.

To run the HPCC benchmark, first create the HPL input file and then simply exeute the hpcc command from cli.

Install and use IOZONE:

```
$ sudo apt istall iozone
```

IOzone performs the following 13 types of test. If you are executing iozone test on a database server, you can focus on the 1st 6 tests, as they directly impact the database performance.

* Read – Indicates the performance of reading a file that already exists in the filesystem.
* Write – Indicates the performance of writing a new file to the filesystem.
* Re-read – After reading a file, this indicates the performance of reading a file again.
* Re-write – Indicates the performance of writing to an existing file.
* Random Read – Indicates the performance of reading a file by reading random information from the file. i.e this is not a sequential read.
* Random Write – Indicates the performance of writing to a file in various random locations. i.e this is not a sequential write.
* Backward Read
* Record Re-Write
* Stride Read
* Fread
* Fwrite
* Freread
* Frewrite

IOZONE can be run in parallel over multiple threads, and use different output files size to stress performance.

```
$ ./iozone -a -b output.xls
```

Executes all stests and create an XLS output to simplify the analysis of the results.

Here you will find an introduction to IOZONE with some examples: https://www.cyberciti.biz/tips/linux-filesystem-benchmarking-with-iozone.html



### Test slurm cluster

 * Run a simple MPI program on the cluster
 * Run an interactive job
 * Use the OSU ping pong benchmark to test the VM interconnect.
 
Install OSU MPI benchmarks: download the latest tarball from http://mvapich.cse.ohio-state.edu/benchmarks/.


```
$ tar zxvf osu-micro-benchmarks-7.3.tar.gz

$ cd osu-micro-benchmarks-7.3/

$ sudo apt install make g++-12

$ sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-12 100
$ sudo update-alternatives --install /usr/bin/c++ c++ /usr/bin/g++-12 100
$ ./configure CC=/usr/bin/mpicc CXX=/usr/bin/mpicxx --prefix=/shared/OSU/
$ make
$ make install
```

## References

[1] Configure DNSMASQ https://computingforgeeks.com/install-and-configure-dnsmasq-on-ubuntu/?expand_article=1

[2] Configure NFS Mounts https://www.howtoforge.com/how-to-install-nfs-server-and-client-on-ubuntu-22-04/

[3] Configure network with netplan https://linuxconfig.org/netplan-network-configuration-tutorial-for-beginners

[4] SLURM Quick Start Administrator Guide https://slurm.schedmd.com/quickstart_admin.html

[5] Simple SLURM configuration on Debian systems: https://gist.github.com/asmateus/301b0cb86700cbe74c269b27f2ecfbef

[6] HPC Challege https://hpcchallenge.org/hpcc/

[7] IO Zone Benchmarks https://www.iozone.org/
