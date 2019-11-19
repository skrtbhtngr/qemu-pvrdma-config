Currently using a **nested virtualization** setup where 2 VMs running on a physical machine are the (hypervisor) hosts. Two nested VMs run on top of these, which are the guest VMs.

### Host VM base config:
* 8 GB RAM
* 8 CPU cores
* 100 GB Disk
* Fedora 31 Workstation
    * default partitioning
    * latest vendor kernel

### Guest VM base config:
* 2 GB RAM
* 2 CPU cores
* 20 GB Disk
* Fedora 29 Workstation
    * default partitioning
    * latest vendor kernel

## Commands to run on the hosts after base installation

#### Update system/Install necessary packages

```
dnf update

dnf groupinstall -y "Development Tools"

dnf install -y vim neovim ctags clang htop flex bison ncurses-devel elfutils-libelf-devel openssl-devel libtool rpcgen gnutls-devel libxml2-devel libtirpc-devel device-mapper-devel yajl-devel glib-devel libnl3-devel libpciaccess-devel libvirt virt-manager gtk3-devel cmake libudev-devel valgrind-devel ninja-build python3-devel python3-Cython rdma-core-devel libibverbs-utils virt-install python git-email gnome-tweak-tool yum-utils terminator tigervnc-server
```

#### Add NFS export for the vdisks

```
systemctl enable nfs-server

mkdir /share        # vdisks are stored here

cat > /etc/exports <<EOF
/share  <IP_A>(rw,no_root_squash) \
        <IP_B>(rw,no_root_squash) \
        <HOSTNAME_A>(rw,no_root_squash) \
        <HOSTNAME_B>(rw,no_root_squash)
EOF

exportfs -r
```

#### Configure QEMU

```
cd ~
git clone https://github.com/skrtbhtngr/qemu.git
cd qemu
git submodule init
git submodule update --recursive
mkdir build
cd build
../configure --enable-trace-backends=log,simple,syslog --enable-debug --enable-rdma --enable-pvrdma --enable-gtk --target-list=x86_64-softmmu --disable-nettle
make -j8 > /dev/null
make rdmacm-mux

cat > /etc/qemu-ifup <<EOF
#!/bin/sh
set -x

switch=br0

if [ -n "$1" ];then
        # tunctl -u `whoami` -t $1 (use ip tuntap instead!)
        ip tuntap add $1 mode tap user `whoami`
        ip link set $1 up
        sleep 0.5s
        # brctl addif $switch $1 (use ip link instead!)
        ip link set $1 master $switch
        exit 0
else
        echo "Error: no interface specified"
        exit 1
fi
EOF
```

#### Setup network (for macvtap)

```
nmcli con add type ethernet ifname ens18
nmcli con modify ethernet-ens18 ipv4.addresses x.x.x.x
nmcli con modify ethernet-ens18 ipv4.gateway x.x.x.x
nmcli con modify ethernet-ens18 ipv4.dns x.x.x.x
nmcli con modify ethernet-ens18 ipv4.dns-search x.x.x
nmcli con modify ethernet-ens18 ipv4.method manual
nmcli con down ethernet-ens18
nmcli con up ethernet-ens18
```

#### Setup network (for bridge)

```
nmcli con add type bridge ifname br0
nmcli con add type bridge-slave ifname enp1s0 master br0
nmcli con modify bridge-br0 bridge.stp no
nmcli con modify bridge-br0 ipv4.addresses x.x.x.x
nmcli con modify bridge-br0 ipv4.gateway x.x.x.x
nmcli con modify bridge-br0 ipv4.dns x.x.x.x
nmcli con modify bridge-br0 ipv4.dns-search x.x.x
nmcli con modify bridge-br0 ipv4.method manual
nmcli con up bridge-br0
```

#### Configure kernel modules

```
cd /lib/modules/5.x.x/kernel/drivers/infiniband/core/
sudo mv rdma_cm.ko ib_cm.ko ~/5.x.x/

cat > /etc/modules-load.d/rdma.conf <<EOF
rdma_rxe
EOF

systemctl enable systemd-modules-load.service
```


#### Configure libvirt

```
sed -i.bkp 's/^#dynamic_ownership/dynamic_ownership/g' /etc/libvirt/qemu.conf
sed -i.bkp 's/^#clear_emulator_capabilities = 1/clear_emulator_capabilities = 0/g' /etc/libvirt/qemu.conf
sed -i.bkp 's/^#user/user/g' /etc/libvirt/qemu.conf
sed -i.bkp 's/^#group/group/g' /etc/libvirt/qemu.conf

cat >> /etc/libvirt/qemu.conf <<EOF
cgroup_device_acl = [
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm",
    "/dev/rtc","/dev/hpet",
   "/dev/infiniband/rdma_cm",
   "/dev/infiniband/issm0",
   "/dev/infiniband/issm1",
   "/dev/infiniband/umad0",
   "/dev/infiniband/umad1",
   "/dev/infiniband/uverbs0"]
EOF

sed -i.bkp 's/^#max_backups = 3/max_backups = 10/g' /etc/libvirt/virtlogd.conf

systemctl enable virtlogd.service
```

#### Misc. system settings

```
systemctl enable sshd

In /etc/selinux/config:
SELINUX=disabled


sudo visudo
skrtbhtngr    ALL=(ALL)       NOPASSWD:ALL


cat > /root/.bashrc <<EOF
alias add_rxe='echo ens18 > /sys/module/rdma_rxe/parameters/add'
alias rdma_mux='/home/skrtbhtngr/qemu/build/rdmacm-mux -d rxe0'
alias test_rxe='if [ -d /sys/class/infiniband/rxe0 ]; then echo "rxe0 exists!"; fi'
EOF

```