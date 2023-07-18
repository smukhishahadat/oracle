# How to Install Oracle 11g on Ubuntu Linux: Complete Guide

Oracle Database is a reliable multiplatform database management software developed by Oracle Corporation. It offers a set of advantages, which include high performance, wide functionality, clusters support, PL/SQL support etc. Oracle Database (will be referred to as Oracle in this blog post) can be installed on Solaris, Windows and Linux. Installation on Windows is the easiest option while installing Oracle on Solaris and Linux needs manual pre-installation configuration. As for Linux distributions, Oracle recommends to install Oracle database on Red-Hat, Oracle Linux, and SUSE Linux Enterprise Server.

However, you can install Oracle on Ubuntu Linux and Open SUSE Linux as well. Ubuntu is a popular Linux distribution that is widely used in the world and today’s blog post provides step-by-step instructions of how to install Oracle on Ubuntu. In this tutorial, we will install Oracle database Enterprise Edition (EE) 11g R2 on Ubuntu.

Preparing Ubuntu Linux
In this tutorial, a freshly installed Ubuntu 16.04.6 is used. The operating system was installed on the VMware VM without enabling the “Download updates while installing Ubuntu” option and without using the “Install third-party software” option during OS deployment.

If you have never installed Oracle on Linux before, you should learn how to install Oracle on Ubuntu on a virtual machine before installing Oracle on a physical computer or a server.

If you use a VMware VM, don’t forget to install VMware Tools. You can also use VirtualBox and Hyper-V virtual machines.

Make sure that you have selected the correct time zone during installation. The time zone is GMT 0 (London) in this example.

You can check the time zone with the command:

timedatectl

Check the Ubuntu version:

lsb_release -a

The output is the following in our example:

Installing Oracle database on Ubuntu 16 – checking the version of Ubuntu

In this tutorial about how to install Oracle 11g on Ubuntu, the following VM hardware parameters are used:

CPU: 1 CPU
RAM: 4 GB
Hard disk drive: 40 GB
user1 is a regular user in this example (user1 is created during Ubuntu installation and is used to log into Ubuntu).

The following packages are installed:

vim, the text editor
net-tools, tools for network management
openssh-server, the SSH server that allows you to connect to the Linux console (terminal) remotely
If you don’t have these packages installed in your Ubuntu Linux, you can install them by using the command:

sudo apt-get install -y vim net-tools openssh-server

The SSH server is installed for convenience. You should run many commands in the console before using the Oracle installer on Ubuntu.

In order to enable authentication by using passwords, edit the sshd_config file:

vim /etc/ssh/sshd_config

Uncomment the line (remove “#” at the beginning of a line):

PasswordAuthentication yes

Save changes and quit by pressing the Esc key in vim and typing :wq

Restart the sshd service (daemon), by running the command with root privileges:

sudo service ssh restart

Configuring a Swap File
Ubuntu 16 uses a swap partition by default. You can customize the size of the swap partition when installing Ubuntu. If you have selected the default partitioning options or selected the incorrect size of the swap partition, you can still create a swap file of the custom size and use the swap file instead of an existing swap partition. The swap size should be equal to twice the RAM size. If you are going to install Oracle on Ubuntu with 4 GB of RAM, you should prepare a swap file or partition that is 8 GB. Let’s create the 8-GB swap file.

Temporary disable using the swap area:

sudo swapoff -a

Create a new 8-GB file with the dd tool:

sudo dd if=/dev/zero of=/swapfile bs=1G count=8

Set the created 8-GB file to be used as a swap file:

sudo mkswap /swapfile

Enable using swap again:

sudo swapon /swapfile

Check the size of your swap area after creating a new swap file:

grep SwapTotal /proc/meminfo

Configuring Network Settings
By default, Ubuntu 16 obtains an IP address automatically for a network interface (if a DHCP server is present in the network). You need to set a static IP address and configure a hostname before you can install Oracle on Ubuntu.

Check the current IP configuration:

ifconfig

In the output, you can see the names of your network adapters and IP addresses. In our case, the name of the needed network interface is ens33.

Configuring the static IP address
Edit the network interfaces configuration file with vim:

sudo vim /etc/network interfaces

The static IP address needed for installing Oracle database on Ubuntu is 192.168.101.11 on the current interface of the Linux machine in this example. Edit the configuration file as follows:

# interfaces(5) file used by ifup(8) and ifdown(8)

auto lo

iface lo inet loopback

auto ens33

iface ens33 inet static

address 192.168.101.11

netmask 255.255.255.0

gateway 192.168.101.2

dns-nameservers 192.168.101.2 8.8.8.8

Save changes and quit.

Apply the changes:

sudo /etc/init.d/networking restart

Or reboot the machine:

init 6

Verify that the new IP settings are applied:

ifconfig

Or:

hostname -I

After checking the IP address configuration, check the hostname.

Checking the hostname
In our case, the hostname used for installing Oracle on Ubuntu is ubuntu-oracle11.

Check the current hostname:

hostnamectl

If you have forgotten to configure the hostname during the installation of Ubuntu or want to change the hostname for other reasons, you can still do it.

In order to rename the host and set a new hostname (ubuntu-oracle11), run the command:

sudo hostnamectl set-hostname ubuntu-oracle11

Verify whether the new host name is applied:

less /etc/hostname

Edit the hosts file:

sudo vim /etc/hosts

The contents of the /etc/hosts file must look like this:

127.0.0.1 localhost

127.0.1.1 ubuntu-oracle11

Restart the machine:

init 6

Try to ping the defined hostname on your Ubuntu machine:

ping ubuntu-oracle11

Configuring the environment for Oracle
To install Oracle on Ubuntu, you need to configure the Linux environment: create system users and groups, configure kernel parameters, set up a system profile, set environment variables for a user, define shell limits, create symlinks, and install required packages.

Creating users and groups
Open the local console or connect to the Linux console via SSH as a regular user and then get the root privileges:

sudo -i

The below commands must be executed as root.

Add the groups required by Oracle.

Create the Oracle Inventory group:

groupadd oinstall

Create the Oracle DBA group:

groupadd dba

Create the home directory for the Oracle user:

mkdir /home/oracle/

Create the directory for installing Oracle:

mkdir -p /u01/app/oracle

Then create the Oracle user account that is a member of the dba group, has the /home/oracle/ home directory and uses /bin/bash as the default shell:

useradd -g oinstall -G dba -d /home/oracle -s /bin/bash oracle

Set the password for the oracle user (don’t forget this password):

passwd oracle

Set the oracle user as the owner of the Oracle home directory and Oracle installation directory. The oracle user is a member of the oinstall group.

chown -R oracle:oinstall /home/oracle

chown -R oracle:oinstall /u01/app/oracle

Note: Unlike Oracle 10, creating the nobody group is not needed for Oracle 11.

Create the directory for Oracle Inventory:

mkdir -p /u01/app/oraInventory

Set the oracle user as the owner for the Oracle Inventory directory:

chown -R oracle:oinstall /u01/app/oraInventory

Configuring kernel parameters
Custom kernel parameters are required to install Oracle on Ubuntu and these kernel parameters affect Oracle performance. Shared memory parameters, including minimum and maximum size, the number of shared memory segments, and semaphores that can be considered as flags for shared memory must be customized according to the Oracle documentation. You also need to set the maximum allowed number of files that can be opened at once, the maximum number of simultaneous network connections, the size of network send and receive buffer. The considered kernel parameters can be categorized in three groups: kernel specifics (kernel), network specifics (network), and file handlers (fs). Edit the /etc/sysctl.conf file to override the parameters of the Linux kernel:

vim /etc/sysctl.conf

Append the lines shown below to the end of this configuration file.

# ============================

# Oracle 11g

# ============================

# semaphores: semmsl, semmns, semopm, semmni

kernel.sem = 250 32000 100 128

kernel.shmall = 2097152

kernel.shmmni = 4096

# Replace kernel.shmmax with the half of your memory size in bytes

# if lower than 4 GB minus 1

# 2147483648 is 2 GigaBytes (4 GB of RAM / 2)

kernel.shmmax=2147483648

#

# Max number of network connections. Use sysctl -a | grep ip_local_port_range to check.

net.ipv4.ip_local_port_range = 9000  65500

#

net.core.rmem_default = 262144

net.core.rmem_max = 4194304

net.core.wmem_default = 262144

net.core.wmem_max = 1048576

#

# The maximum allowed value, set to avoid overhead and input/output errors

fs.aio-max-nr = 1048576

# 512 * Processes

fs.file-max = 6815744

fs.suid_dumpable = 1

#

# To allow dba to allocate hugetlbfs pages

# 1001 is your oinstall group, you can check this id with the grep oinstall /etc/group command

vm.hugetlb_shm_group = 1001

Apply the kernel parameters you have set:

sysctl -p

Configuring shell limits
You have to set shell limits to improve performance of the Oracle database software.

Edit the /etc/security/limits.conf file:

vim /etc/security/limits.conf

Add the following lines to the end of the configuration file:

# Oracle

oracle           soft    nproc   2047

oracle           hard    nproc   16384

oracle           soft    nofile  1024

oracle           hard    nofile  65536

oracle           soft    stack   10240

The first column specifies the user for whom the limits are set.

The second column operates with two options—soft and hard. Soft is the maximum number that can be set by the user, while hard is the maximum number that can be reconfigured by the defined user (oracle). If the oracle user reaches the critical low value of 2047 set in the first line and needs to extend the limit, the user can increase the limit to 16384. Values higher than 16384 cannot be set by the oracle user but can be set by root.

The third column defines on which resources the set limits are applied.

The fourth column defines the maximum number of the resource parameter specified in the third column.

PAM configuration
Ensure that the /etc/pam.d/login configuration file contains the following line:

session    required   pam_limits.so

You can do it with the command:

cat /etc/pam.d/login | grep pam_limits.so

If the mentioned line is missing, add this line manually.

Setting the shell profile
System wide environmental variables are set in the global bash shell profile that is set in the /etc/profile configuration file.

Edit /etc/profile and set some needed parameters for oracle globally and permanently:

vim /etc/profile

Add the following lines:

if [ $USER = “oracle” ]; then

        if [ $SHELL = “/bin/ksh” ]; then

              ulimit -p 16384

              ulimit -n 65536

        else

              ulimit -u 16384 -n 65536

        fi

fi

Note: It is useful to know when each shell configuration file can be used because later you will need to configure a profile that contains environment variables for the oracle user.

Bash login shell loads its configuration from the following files in the appropriate order:

/etc/profile
~/.bash_profile
~/.bash_login
~/.profile
/etc/profile can be considered as a global profile for the bash login shell.

./bash_profile is applied for bash login shells, for example, when you log into the command line interface of Linux directly by using a keyboard connected to this Linux computer, or when opening a new console session after you log in by using the SSH terminal.

.profile – this file is read for bash and other shells like sh.

Bash non-login interactive shell loads configuration from ~/.bashrc

It means that if you are already logged into your Linux machine (for example you have logged into Ubuntu with GUI) and then opened a new console (terminal) window, then shell configuration, including environment variables, is loaded from the .bashrc file before you get access to the command prompt.

Bash non-login and non-interactive shell loads configuration that is set in the $BASH_ENV environment variable. The non-login and non-interactive shell is used when a script runs.

Installing the required packages
You need to install packages required by Oracle. Take care when installing them because a missing package may cause errors during Oracle installation or database creation.

Update the repository tree:

apt-get update

Install packages to avoid errors that can occur during the installation of Oracle on Ubuntu. Most packages can be installed with Ubuntu standard package manager from online repositories.

apt-get install alien

apt-get install autoconf

apt-get install automake

apt-get install autotools-dev

apt-get install binutils

apt-get install bzip2

apt-get install doxygen

apt-get install elfutils

apt-get install expat

apt-get install gawk

apt-get install gcc

apt-get install gcc-multilib

apt-get install g++-multilib

apt-get install libelf-dev

apt-get install libltdl-dev

apt-get install libodbcinstq4-1 libodbcinstq4-1:i386

apt-get install libpth-dev

apt-get install libpthread-stubs0-dev

apt-get install libstdc++5

apt-get install make

apt-get install openssh-server

apt-get install rlwrap

apt-get install rpm

apt-get install sysstat

apt-get install unixodbc

apt-get install unixodbc-dev

apt-get install unzip

apt-get install x11-utils

apt-get install zlibc

apt-get install libaio1

apt-get install libaio-dev

There are a few packages left to install, but to install them you’ll need these tips and tricks.

apt-get install ia32-libs

When you run the command to install this package with apt-get, you will see a message saying that this package is not available.

There is an alternative package that can be installed instead of ia32-libs. Install the lib32z1 alternative package:

apt-get install lib32z1

Let’s install the next package:

apt-get install libmotif4

This package is also not available. You should download and install this package manually. Download the libmotif4_2.3.4-8ubuntu1_amd64.deb file from a free resource. Save the downloaded deb file to a custom directory, for example, /home/user1/Downloads, before you can install this package.

The workflow of installing deb packages in Ubuntu is the following (execute commands with root privileges):

sudo dpkg -i package_name.deb

sudo apt-get -f install

In this case, we run the commands for installing packages as a regular user, for example, user1 by using sudo. Press Ctrl+D to exit the root session and to turn back to the user1 session in the console (terminal).

Go to the directory where the downloaded files are stored:

cd /home/user1/Downloads

Download the file by using a direct link:

wget http://launchpadlibrarian.net/207968936/libmotif4_2.3.4-8ubuntu1_amd64.deb

sudo dpkg -i libmotif4_2.3.4-8ubuntu1_amd64.deb

sudo apt-get -f install

The next package cannot be installed automatically:

sudo apt-get install libpthread-stubs0

E: Unable to locate package libpthread-stubs0

Download and install this package manually.

wget http://launchpadlibrarian.net/154418307/libpthread-stubs0-dev_0.3-4_amd64.deb

sudo dpkg -i libpthread-stubs0-dev_0.3-4_amd64.deb

sudo apt-get -f install

Similarly install the next package:

sudo apt-get install lsb-cxx

E: Unable to locate package lsb-cxx

You can download the lsb-cxx package manually.

wget http://packages.linuxmint.com//pool/upstream/l/lsb/lsb-cxx_4.1+Debian11ubuntu6mint1_amd64.deb

sudo dpkg -i lsb-cxx_4.1+Debian11ubuntu6mint1_amd64.deb

sudo apt-get -f install

Install one more package:

sudo apt-get install pdksh

E: Package ‘pdksh’ has no installation candidate

The workflow for installing the pdksh package is the same. This package can be downloaded here.

wget http://launchpadlibrarian.net/200019501/pdksh_50e-2ubuntu1_all.deb

sudo dpkg -i pdksh_50e-2ubuntu1_all.deb

sudo apt-get -f install

Oracle 11g requires the installation of the 32-bit version of the libstdc++5 package that is not installed on Ubuntu 16 automatically. Here’s how you can install this package manually.

The next commands are executed as root. Create a temporary directory to download the package:

mkdir /tmp/libstdc++5

cd /tmp/libstdc++5

Download the package (links to the 32-bit and 64-bit packages are shown):

wget http://mirrors.edge.kernel.org/ubuntu/pool/universe/g/gcc-3.3/libstdc++5_3.3.6-30_i386.deb

wget http://mirrors.edge.kernel.org/ubuntu/pool/universe/g/gcc-3.3/libstdc++5_3.3.6-30_amd64.deb

Install the downloaded packages with dpkg:

dpkg –force-architecture -i libstdc++5_3.3.6-30_i386.deb

Note: If links don’t work, newer versions of packages can be published instead of older versions. In this case, visit a web page in the browser and copy the correct links to the needed deb packages:

http://mirrors.edge.kernel.org/ubuntu/pool/universe/g/gcc-3.3/

Reboot the Linux machine to apply the changes.

init 6

Creating symbolic links
You have to create symbolic links (symlinks) to make the file and directory structure used in Ubuntu similar to the file structure in Red Hat. Create the symlinks to make a file system structure appear identical to the Red Hat file system structure to prevent errors during Oracle installation.

Run commands as root:

mkdir /usr/lib64

ln -s /etc /etc/rc.d

ln -s /usr/bin/awk /bin/awk

ln -s /usr/bin/basename /bin/basename

ln -s /usr/bin/rpm /bin/rpm

ln -s /lib/x86_64-linux-gnu/libgcc_s.so.1 /usr/lib64/

ln -s /usr/lib/x86_64-linux-gnu/libc_nonshared.a /usr/lib64/

ln -s /usr/lib/x86_64-linux-gnu/libpthread_nonshared.a /usr/lib64/

ln -s /usr/lib/x86_64-linux-gnu/libstdc++.so.6 /usr/lib64/

ln -s /usr/lib/x86_64-linux-gnu /usr/lib64

ln -sf /bin/bash /bin/sh

ln -s /etc/rc0.d /etc/rc.d/rc0.d

ln -s /etc/rc2.d /etc/rc.d/rc2.d

ln -s /etc/rc3.d /etc/rc.d/rc3.d

ln -s /etc/rc4.d /etc/rc.d/rc4.d

ln -s /etc/rc5.d /etc/rc.d/rc5.d

ln -s /etc/rc6.d /etc/rc.d/rc6.d

ln -s /etc/init.d /etc/rc.d/init.d

The above commands help prevent errors, such as:

genclntsh: Failed to link libclntsh.so.11.1 in make file for rdbms/lib/ins_rdbms.mk because of missing library: /usr/bin/ld: cannot find /usr/lib64/libpthread_nonshared.a inside
lib//libagtsh.so: undefined reference to `nnfyboot’ in make: rdbms/lib/dg4odbc] Error 1
Now let’s prevent one more error: /lib64/libgcc_s.so.1: File or directory does not exist, while creating lib/liborasdkbase.so.11.1 in ins_rdbms.mk. Go to the /lib64 directory and execute the command:

cd /lib64

ln -s /lib/x86_64-linux-gnu/libgcc_s.so.1 .

Don’t miss the dot at the end of the command.

Set the Linux version as Red Hat Linux release 5 in /etc/redhat-release. Red Hat Linux distributions store their version in that file.

echo ‘Red Hat Linux release 5’ > /etc/redhat-release

Configuring the oracle user profile
Now you need to configure the user environment for the oracle user.

Login as oracle if you have opened the console as another user:

su oracle

Run the commands as oracle:

cd

vim ~/.bashrc

Add the following lines to the .bashrc file:

# Oracle Settings

TMP=/tmp; export TMP

TMPDIR=$TMP; export TMPDIR

# Enter your hostname

ORACLE_HOSTNAME=ubuntu-oracle11; export ORACLE_HOSTNAME

ORACLE_UNQNAME=ORADB11G; export ORACLE_UNQNAME

ORACLE_BASE=/u01/app/oracle; export ORACLE_BASE

ORACLE_HOME=$ORACLE_BASE/product/11.2.0/dbhome_1; export ORACLE_HOME

ORACLE_SID=ORADB11G; export ORACLE_SID

PATH=/usr/sbin:$PATH; export PATH

PATH=$ORACLE_HOME/bin:$PATH; export PATH

LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib:/usr/lib64; export LD_LIBRARY_PATH

CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib; export CLASSPATH

umask 022

Apply the settings:

source ~/.bashrc

You should restart Ubuntu now. Note that settings will be applied after reboot even without running source ~/.bashrc.

init 6
