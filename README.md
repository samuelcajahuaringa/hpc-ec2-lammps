# Building a HPC Slurm Cluster using Amazon EC2

In this repository, I document my endeavor to construct a scalable
high-performance computing (HPC) cluster using [Amazon
ec2](https://aws.amazon.com/ec2/) instantces, specifically tailored for
scientific applications. The knowledge gained from this tutorial aims to
assist you to understand the basis principles of building your own
functional HPC cluster, management using
[Slurm](https://slurm.schedmd.com/), and finally as example we run a
parallel aplication the [LAMMPS](https://www.lammps.org/) software.

## 1. Set-up machine that will form cluster:

1.1 Getting the AWS EC2 instances up and running

------------------------------------------------------------------------

-   instantces a Linux virtual machine (VM) withing Amazon EC2 and how
    to connect to it.

-   The operating system we use in this project is **Ubuntu Server 20.04
    LTS**, which has a free tier AMI available in Amazon EC2.

-   Use **t2.micro** instance type to make sure you are not incurring
    any cost.

-   Use **default VPC network** when asked for VPC.

-   Do not add any extra storage. The default storage is more than
    enough for our purpose.

-   Connect all VMs to the same Security Group at the time of instantiating them, otherwise, they can't access each other.

    :   -   For the first VM, choose **create new security group**
        -   Add one rule with the type of **All traffic** from source
            0.0.0.0/0
        -   Add second rule with the type to **NFS** and source to your
            security group.

-   For the rest of VMs, choose **select an existing security group**
    and select the security group that you have created for the first VM

### 1.2 Give your VMs easy names

Use the **ifconfig** command to know your VM's private IP address (it
should be in the range of 172.x.x.x). You can also find it in the
**botton** of the AWS web console for each VM.

Set the **hostname** of each VM by altering /etc/hostname:

``` bash
sudo vim /etc/hostname
```

In his example was used 3 VMs, one for head node and 2 for workers node,
edit the existing name and make it to be head and node01, node02 for the
workers nodes.

Reebot the VMs in order to apply the changes.

On all VMs, edit `/etc/hosts` and add following lines at the end with
each VM's IP address and its name. Remember to use **sudo** for editing.

It should be like for head node:

``` bash
127.0.0.1 localhost
<IP-address of node 01> node01 
<IP-address of node 02> node02 

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts   
```

for the node01

``` bash
127.0.0.1 localhost
<IP-address of head> head
<IP-address of node 02> node02 

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts   
```

for the node02

``` bash
127.0.0.1 localhost
<IP-address of head> head
<IP-address of node 01> node01

# The following lines are desirable for IPv6 capable hosts
::1 ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts   
```

### 1.3 Configure ubuntu user to login to the worker nodes without password

-   On the head node do the following to generate key pairs:

``` bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_25519
cat ~/.ssh/id_25519.pub
```

-   Copy the public key content (make sure that it includes:
    `ssh-ed25519……ubuntu@head`) and then do the following on each remote
    node

``` bash
echo "<paste public key here>" >>  ~/.ssh/authorized_keys
```

-   Now, try to ssh to other nodes from head node, in the following way:

``` bash
ssh -i ~/.ssh/id_25519 node01
ssh -i ~/.ssh/id_25519 node02
```

## 2. Install and configuration of NFS:

### 2.1 Configure NFS server on the Head node

Create a shared directory and set the permission of a directory (Note
that this should be the same across all the nodes).

``` bash
sudo mkdir /shared 
sudo chown nobody.nogroup -R /shared
sudo chmod 777 -R /shared
```

Install the NFS package

``` bash
sudo apt-get update
sudo apt install nfs-kernel-server -y
```

Configure `/etc/exports` to use a new shared directory

``` bash
sudo sh -c 'echo "/shared 172.0.0.0/8(rw,sync,no_root_squash,no_subtree_check)" >> /etc/exports'
sudo exportfs -a
```

should look like this:

``` bash
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/shared 172.0.0.0/8(rw,sync,no_root_squash,no_subtree_check)
```

### 2.2 Configure NFS on the workes nodes

Create a shared directory and set the permission of a directory

``` bash
sudo mkdir /shared
sudo chown nobody.nogroup /shared
sudo chmod -R 777 /shared
```

Install the NFS package

``` bash
sudo apt-get update
sudo apt install nfs-kernel-server -y
```

We want the NFS share to mount automatically when the nodes boot. For
that purpose, edit **/etc/fstab** to accomplish this by adding the
following line at the end (should look like this):

``` bash
LABEL=cloudimg-rootfs   /        ext4   discard,errors=remount-ro       0 1
LABEL=UEFI      /boot/efi       vfat    umask=0077      0 1
<IP-address of head node>:/shared  /shared nfs defaults   0 0
```

Mount the share directory:

``` bash
sudo mount -a
```

If everything is fine, you should see the NFS mount on worker nodes with
**df -h**

``` bash
<IP-addres of head node>:/shared   7941632 2305920   5619328  30% /shared
```

## 3. Combine VMs as a cluster with SLURM job scheduler:

Update OS and preinstalled software to the latest version before
installing SLURM for all nodes.

``` bash
sudo apt update
sudo apt upgrade
```

### 3.1 Install SLURM on head node

``` bash
sudo apt install slurm-wlm -y
```

### 3.2 SLURM configuration head node

We'll use the default SLURM configuration file as a base. Copy it over:

``` bash
cd /etc/slurm
sudo cp /usr/share/doc/slurm-client/examples/slurm.conf.simple.gz .
sudo gzip -d slurm.conf.simple.gz
sudo mv slurm.conf.simple slurm.conf
```

Then edit `/etc/slurm-llnl/slurm.conf`

#### 3.2.1 Set the cluster name

This is somewhat superficial, but you can customize the cluster name in
the "LOGGING AND ACCOUNTING" section:

``` bash
ClusterName=cluster
```

#### 3.2.2 Set the control machine info

Modify the first configuration line to include the hostname of the
master node, and its IP address:

``` bash
SlurmctldHost=head(<ip addr of head node>)
```

#### 3.2.3 Set authentication

All communications between Slurm components are authenticated. The
authentication infrastructure is provided by a dynamically loaded plugin
chosen at runtime via the **AuthType** keyword in the Slurm
configuration file.

``` bash
AuthType=auth/munge
```

#### 3.2.4 Customize the scheduler algorithm

SLURM can allocate resources to jobs in a number of different ways, but
for our cluster we'll use the **consumable resources** method. This
basically means that each node has a consumable resource (in this case,
CPU cores), and it allocates resources to jobs based on these resources.
So, edit the SelectType field and provide parameters, like so:

``` bash
SelectType=select/cons_res
SelectTypeParameters=CR_Core
```

#### 3.2.5 Add the nodes

Now we need to tell SLURM about the compute nodes. Near the end of the
file, there should be an example entry for the compute node. Delete it,
and add the following configurations for the cluster nodes:

``` bash
NodeName=head NodeAddr=<ip addr head> CPUs=1 State=UNKNOWN
NodeName=node01 NodeAddr=<ip addr node01> CPUs=1 State=UNKNOWN
NodeName=node02 NodeAddr=<ip addr node01> CPUs=1 State=UNKNOWN
```

#### 3.2.6 Create a partition

SLURM runs jobs on 'partitions,' or groups of nodes. We'll create a
default partition and add our 2 compute nodes to it. Be sure to delete
the example partition in the file, then add the following on one line:

``` text
PartitionName=test Nodes=node[01-02] Default=YES MaxTime=INFINITE State=UP
```

### 3.3 Configure cgroup support

The latest update of SLURM brought integrated support for cgroups kernel
isolation, which restricts access to system resources. We need to tell
SLURM what resources to allow jobs to access. To do this, create the
file `sudo touch /etc/slurm/cgroup.conf` with following information:

``` text
CgroupMountpoint="/sys/fs/cgroup"
CgroupAutomount=yes
CgroupReleaseAgentDir="/etc/slurm/cgroup"
AllowedDevicesFile="/etc/slurm/cgroup_allowed_devices_file.conf"
ConstrainCores=no
TaskAffinity=no
ConstrainRAMSpace=yes
ConstrainSwapSpace=no
ConstrainDevices=no
AllowedRamSpace=100
AllowedSwapSpace=0
MaxRAMPercent=100
MaxSwapPercent=100
MinRAMSpace=30
```

Now, whitelist system devices by creating the file
`sudo touch /etc/slurm/cgroup_allowed_devices_file.conf` with this
information:

``` text
/dev/null
/dev/urandom
/dev/zero
/dev/sda*
/dev/cpu/*/*
/dev/pts/*
/shared*   
```

Note that this configuration is pretty permissive, but for our purposes,
this is okay. You could always tighten it up to suit your needs.

### 3.4 Copy the Configuration Files to Shared Storage

In order for the other nodes to be controlled by SLURM, they need to
have the same configuration file, as well as the Munge key file. Copy
those to shared storage to make them easier to access, like so:

``` bash
sudo cp slurm.conf cgroup.conf cgroup_allowed_devices_file.conf /shared
sudo cp /etc/munge/munge.key /shared
```

### 3.5 Enable and Start SLURM Control Services

Munge:

``` bash
sudo systemctl enable munge
sudo systemctl start munge
```

The SLURM daemon:

``` bash
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

And the control daemon:

``` bash
sudo systemctl enable slurmctld
sudo systemctl start slurmctld
```

### 3.6 Reboot (optional)

This step is optional, but if you are having problems with Munge
authentication, or your nodes can't communicate with the SLURM
controller, try rebooting it.

## 4. SLURM configuration workers node

Copy the configuration files: The configuration on all the worker nodes
should match the configuration on the head node. So, copy it through the
shared storage

``` bash
sudo cp /shared/munge.key /etc/munge/munge.key
sudo cp /shared/slurm.conf /etc/slurm/slurm.conf
sudo cp /shared/cgroup* /etc/slurm
```

### 4.1 Enable and Start Munger

``` bash
sudo systemctl enable munge
sudo systemctl start munge
```

### 4.2 Enable and Start SLURM

``` bash
sudo systemctl enable slurmd 
sudo systemctl start slurmd
```

## 5. Test your SLURM cluster

Now that we've configured the SLURM controller and each of the nodes, we
can check to make sure that SLURM can see all of the nodes by running
sinfo on the master node (a.k.a. "the head node"):

``` bash
ubuntu@head:~$ sinfo 
PARTITION AVAIL  TIMELIMIT   NODES STATE NODELIST
test*        up   infinite       2  idle node[01-02]
```

Now we can run a test job by telling SLURM to give us 2 nodes, and run
the hostname command on each of them:

``` bash
srun --nodes=2 hostname
```

If all goes well, we should see something like:

``` bash
node01
node02
```

## 6. Installing and Run LAMMPS

To show how the Amzon EC2 cluster can be used to run scientific
calculations, we now install one of the most popular packages for
moleculer dynamics simulations the Large-scale Atomic/Molecular
Massively Parallel Simulator program from Sandia National Laboratories.
LAMMPS makes use of Message Passing Interface (MPI) for parallel
communication and is free and open-source software, distributed under
the terms of the GNU General Public License.

In our cluster configuration, only worker nodes execute computation and
the master node orchestrates jobs. To run MPI programs on this cluster,
you need to ensure that MPI is installed on nodes workers.

### 6.1 Intall c++ compiler

On the head node

``` bash
sudo su -
srun --nodes=2 sudo apt install g++ -y
```

the `sudo su -` to access as root of head node. This will run the
`apt install g++ -y` command on workers nodes in the cluster (change the
2 to match your setup).

Check it was correctly installed on each workers

``` bash
srun --nodes=2 g++ --version
```

### 6.2 Intall cmake

On the head node

``` bash
srun --nodes=2 sudo apt install cmake -y
```

Check it was correctly installed on each workers

``` bash
srun --nodes=2 cmake --version
```

### 6.3 Download LAMMPS

On the head node

``` bash
cd /shared/
wget https://download.lammps.org/tars/lammps-stable.tar.gz
tar -xzf lammps-stable.tar.gz 
```

### 6.4 Compiling LAMMPS

Because we need to compiler LAMMPS, we're going to grab a shell instance
from one of the workers nodes:

``` bash
ubuntu@head:~$ sudo su -
root@head:~# srun --pty bash
root@node01:~# cd /shared/lammps-2Aug2023/src
root@node01:/shared/lammps-2Aug2023/src# make lmpi
```

If all goes well, we should create the lammps executavel `lmp_mpi`.

### 6.5 Runing LAMMPS

On the head node, we test the lennard-jones example. Now that we have
our LAMMPS program, we will create a submission script to run our jobs.
Create the file `/shared/lammps-2Aug2023/examples/melt/job.sh`

``` bash
#!/bin/bash
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=1
#SBATCH --partition=test
#SBATCH --output=slurm.out
#SBATCH --error=slurm.err

cd $SLURM_SUBMIT_DIR

mpirun -np 2 /shared/lammps-2Aug2023/src/lmp_mpi -in in.melt
```

We now have everything we need to run our job

``` bash
cd /shared/lammps-2Aug2023/examples/melt/
sbatch job.sh
Submitted batch job 18
```

If all goes well, we should see the create files

``` bash
log.lammps
slurm.out
slurm.err 
```

Check the outputs files log.lammps and log.8Apr21.melt.g++.4 that is
found in this directory, both must be show the same thermodynamic
information.

### Conclusion

We now have a basically complete cluster. We can run jobs using the
SLURM scheduler; we discussed how to install software; we installed
OpenMPI; and we ran LAMMPS program that use it. Hopefully, your cluster
is functional enough that you can add software and components to it to
suit your projects.

Happy Computing!

Author & Contact:
--------------
Samuel Cajahuaringa - samuelcm@unicamp.br
