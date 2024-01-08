# How to set up a Kubernetes (K8s) Cluster inside Systopya Machines?

## Requirements

* A set of nodes  
* A bridge network (talk with Vilmos about it)

## Hardware and software adopted
 * 4 nodes with Ubuntu 22.04 (leapx19, leapx20, leapx21, leapx22) 
 * 32 VMs with Ubuntu 22.04 
 * VirtualBox 7.0.12

## Install and configure the virtual machines (VMs)

The following softwares need to be install on each node:

* Virtual Box (VB)
* VB Dependencies
* VB Extension Pack

### VB Dependencies

 > sudo apt update -y; sudo apt upgrade -y; sudo apt install -y linux-headers-$(uname -r) dkms;

### VB Installation

```
wget https://download.virtualbox.org/virtualbox/7.0.12/virtualbox-7.0_7.0.12-159484~Ubuntu~jammy_amd64.deb;
sudo dpkg -i namefile.deb;
sudo apt install -f;
VBoxManage --version;
```

## Install VB Extension Pack

The Extension Pack will be used to access virtual machines during installation remotely.

### Download and install the VB Extension Pack and check its installations

```
wget https://download.virtualbox.org/virtualbox/7.0.12/Oracle_VM_VirtualBox_Extension_Pack-7.0.12.vbox-extpack
sudo VBoxManage extpack install Oracle_VM_VirtualBox_Extension_Pack-7.0.12.vbox-extpack
VBoxManage list extpacks
```

## Download the Operating System (OS)

The Ubuntu server 22.02 was used in this documentation. If you use another operating system, modify the commands as necessary.

### Download Ubuntu Server 22.04.3:

wget https://releases.ubuntu.com/22.04.3/ubuntu-22.04.3-live-server-amd64.iso

## Create the VM

### Modify the installation script 

The following script installs an Ubuntu VM called Leapx191 with 6GB of RAM, 2 CPUS, and 60GB of hard disk using a file called ubuntu.iso. It uses two networks, NAT (provides internet) and bridge (provides communication between different hosts). 

Modify the script below as desired. The points to be modified are in italic.

* *VM name* # **Each VM needs its own name.** 
* *OS type* # **You can use: *VBoxManage list ostypes* to see types available.** 
* *Memory size* 
* *Number of CPUs* 
* *Bridge name* # It's the name of the bridge network interface that Vilmos configured.
* *Disk size* 
* *OS filename*

<pre>
#!/bin/bash
MACHINENAME=<i>Leapx191</i> # Name of the VM

# Create a VM and put their files into the same folder
VBoxManage createvm --name $MACHINENAME --ostype "<i>Ubuntu_64</i>" --register --basefolder `pwd`  # OS type

# Set memory and network
VBoxManage modifyvm $MACHINENAME --ioapic on
VBoxManage modifyvm $MACHINENAME <i>--memory 6144 --cpus 2</i> --vram 128 # Memory size and CPUs
VBoxManage modifyvm $MACHINENAME --nic1 nat
VBoxManage modifyvm leapx191 --nic2 bridged <i>--bridgeadapter2 wellisonbridge0</i> # Bridge adapter name

# Create the disk and connect Ubuntu Iso
VBoxManage createhd --filename `pwd`/$MACHINENAME/$MACHINENAME_DISK.vdi <i>--size 61440</i> --format VDI # Disk Size
VBoxManage storagectl $MACHINENAME --name "SATA Controller" --add sata --controller IntelAhci
VBoxManage storageattach $MACHINENAME --storagectl "SATA Controller" --port 0 --device 0 --type hdd --medium  `pwd`/$MACHINENAME/$MACHINENAME_DISK.vdi
VBoxManage storagectl $MACHINENAME --name "IDE Controller" --add ide --controller PIIX4
VBoxManage storageattach $MACHINENAME --storagectl "IDE Controller" --port 1 --device 0 --type dvddrive <i>--medium `pwd`/ubuntu.iso</i> # OS Filename
VBoxManage modifyvm $MACHINENAME --boot1 dvd --boot2 disk --boot3 none --boot4 none

#Enable RDP
VBoxManage modifyvm $MACHINENAME --vrde on
VBoxManage modifyvm $MACHINENAME --vrdemulticon on --vrdeport 10001

#Start the VM
VBoxHeadless --startvm $MACHINENAME
</pre>

### Save the script 

Put the script into a sh file using any editor such as Nano, vim, and so on. 

For instance:
>  nano filename.sh

### Give execute permission to the file

> chmod +x filename.sh
 
### Execute the script

> ./filename.sh

## Access the virtual machine

The VM will be waiting for the OS to be installed. We need to use our computer to access the machine remotely to install the operating system.

### Install Rdesktop to access the VM by Remote Display

> apt -y install rdesktop

### Create an SSH tunnel between your computer and the host.

You will likely be using csremote as a jump host. To make this command easier to access, try configuring the .config file in ssh. Refer to [config](##How-to-configure-the-.config-file) for details.

This command is used when you do not have the ssh .config file configured.

> ssh -L 10001:systopia_node:10001 your_cwl_user@remote.cs.ubc.ca 

This command is used when you do have the ssh .config file configured.

> ssh -L 10001:systopia_node:10001 csremote 

###  Access the VM

> rdesktop -a 16 -N localhost:10001

### Install the VM

Most of these steps are quite easy. 
For instance, choice of keyboard and installer update.
The tricky configuration is the VM bridge network, which will be following detailed:

#### Bridge Network

There are two network interfaces. 
The Host-only network (10.0.2.15) provide internet for the VMs.
The bridge network is the one used to communicate with the K8s Cluster. 
The Host-only network is configured by default. 

To configure the bridge, go to bridge network interface -> edit IPv4 -> manual mode -> enter subnet and IP.

To check the IP range of the bridge network, use the ifconfig command in the systopia_node. You may need to install net-tools to use this command.

```
apt install -y net-tools
ifconfig
```

My host bridge IP is: **172.30.1.19**

Therefore, 

**Subnet**: 172.30.1.0/24 <br>
**VM Bridge IP**: 172.30.1.221 <br>

I choose 172.30.1.221 to a VM called leapx221. You need to choose an IP not yet allocated and the range 1-255.* **The other fields (e.g., DNS and gateway) do not need to be filled in. Finally, save the settings.** 


You also need to check the SSH option to allow remote access after installation. There is no need to install any packages. We will perform manual installations after installing the VM. Wait for the VM to install. 

After the installation and the VM has restarted, stop the VM using control + C if you still have the run terminal. Otherwise, run: 

> VBoxManage controlvm vmname acpipowerbutton

Then, remove the remote virtualization property so the port is not blocked for the next installation.


## Install K8s

I used the Ubuntu PPA method to install K8s because it is simplified. However, PPA repositories are ephemeral. Others alternative to download and install K8s (i.e., kubeadm, kubelet and kubectl) are available on the [official page](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/).


Update the apt package index and install packages needed to use the K8s apt repository

```
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
```

Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so that you can disregard the version in the URL. 

If you want to use a K8s version different than **v1.29**, replace **v1.29** with the desired minor version in the commands below:



> curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg 

Add the appropriate K8s apt repository.

> echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

Update apt package index, then install kubectl, kubeadm and kubelet.

```
sudo apt update
sudo apt install -y kubectl kubeadm kubelet
```


## Install Docker

I used the Ubuntu PPA method to install docker because it is simplified. Other alternatives to download and install Docker are available on the [official page](https://docs.docker.com/engine/install/ubuntu/).

Add Docker's official GPG key.

```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

Add the repository to Apt sources:

```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  
sudo apt-get update
```

> [!NOTE] 
> If you use an Ubuntu derivative distro, such as Linux Mint, you may need to use UBUNTU_CODENAME instead of VERSION_CODENAME.


## How to configure the .config file

You can access lab machines (i.e. desktops) remotely through remote.cs.ubc.ca (for CS grad students), remote.students.cs.ubc.ca (for CS undegrad students), and ssh.ece.ubc.ca (for ECE students). Server machines can then be accessed from the desktop machines. If you don't have access to one of the aforementioned gateways, you are probably not a student or employee, and need to talk to one of the professors to get a guest account.

To make things more convenient, you can set up an ssh config file (usually ~/.ssh/config on Linux-based systems) like this:

<pre>
host csremote
     IdentityFile ~/.ssh/id_rsa
     Hostname     remote.cs.ubc.ca
     User         your_cwl_user

host node_dns_name
     IdentityFile ~/.ssh/id_rsa
     User         node_user
     ProxyJump    csremote
</pre>

For any requests and/or queries, please send an email to the Equipment Tsar Vilmos Soti systopia-equipment@cs.ubc.ca. 



