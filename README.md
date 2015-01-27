# openstack-juno-playbook

Openstackサーバの構築手順 (CentOS7)
============================================================
(環境情報)
[共通]
OS: CentOS7.0
root pw:   openstack123
user id:   openstack
user full: openstack
user pw:   openstack123
NTPサーバ: 192.168.0.1

[host1]
Host: openstack.func.org
eth0: 保守用
      IPAddress: 192.168.0.2
      Netmask:   255.255.255.0
      DefaultGW: 192.168.0.1
      DNS:       192.168.0.1

[guest1]
Host: rdo01.func.org
eth0: 保守用/外部ネットワーク
      IPAddress: 192.168.0.19
      Netmask:   255.255.255.0
      DefaultGW: 192.168.0.1
      DNS:       192.168.0.1
eth1: 内部ネットワーク
      IPAddress: 172.16.10.10
      Netmask:   255.255.255.0

[guest2]
Host: rdo02.func.org
eth0: 保守用/外部ネットワーク
      IPAddress: 192.168.0.21
      Netmask:   255.255.255.0
      DefaultGW: 192.168.0.1
      DNS:       192.168.0.1
eth1: 内部ネットワーク
      IPAddress: 172.16.10.11
      Netmask:   255.255.255.0

[guest3]
Host: rdo03.func.org
eth0: 保守用/外部ネットワーク
      IPAddress: 192.168.0.23
      Netmask:   255.255.255.0
      DefaultGW: 192.168.0.1
      DNS:       192.168.0.1
eth1: 内部ネットワーク
      IPAddress: 172.16.10.12
      Netmask:   255.255.255.0
============================================================

============================================================
(手順)
1. ansibleで設定

2. CentOS 7.0のイメージを/root/CentOS-7.0-1406-x86_64-DVD.isoに配置する。
   # ls /root/CentOS-7.0-1406-x86_64-DVD.iso
   # mount -r -o loop -t iso9660 /root/CentOS-7.0-1406-x86_64-DVD.iso /opt/os/CentOS-70-x86_64

3. ゲストマシンをksでインストールする。
   # virt-install --name=rdo01 --hvm --virt-type kvm --disk path=/var/lib/libvirt/images/rdo01-disk1.img,size=100,sparse=true,cache=none,format=qcow2 --disk path=/var/lib/libvirt/images/rdo01-disk2.img,size=50,sparse=true,cache=none,format=qcow2 --disk path=/var/lib/libvirt/images/rdo01-disk3.img,size=50,sparse=true,cache=none,format=qcow2 --cpu host --vcpus=2 --ram=2048 --network bridge=br0 --network network=internal --os-type=linux --os-variant=rhel7 --location /tmp/CentOS-7.0-1406-x86_64-DVD.iso --noautoconsole --initrd-inject /opt/os/kscfg/rdo01-ks.cfg --extra-args "inst.ks=file:/rdo01-ks.cfg console=tty0 console=ttyS0,115200n8 ksdevice=eth0 ip=192.168.0.19 netmask=255.255.255.0 gateway=192.168.0.1" --graphics vnc,listen=0.0.0.0,port=5901

    # virt-install --name=rdo02 --disk path=/var/lib/libvirt/images/rdo02-disk1.img,size=100,sparse=true,cache=none,format=qcow2 --cpu host --vcpus=2 --ram=2048 --network bridge=br0 --network network=internal --os-type=linux --os-variant=rhel7 --location http://192.168.0.2/os/CentOS-70-x86_64 --noautoconsole --extra-args "ks=http://192.168.0.2/os/kscfg/rdo02-ks.cfg console=tty0 console=ttyS0,115200n8 ksdevice=eth0 ip=192.168.0.21 netmask=255.255.255.0 gateway=192.168.0.1" --graphics vnc,listen=0.0.0.0,port=5902

    # virt-install --name=rdo03 --disk path=/var/lib/libvirt/images/rdo03-disk1.img,size=100,sparse=true,cache=none,format=qcow2 --cpu host --vcpus=2 --ram=2048 --network bridge=br0 --network network=internal --os-type=linux --os-variant=rhel7 --location http://192.168.0.2/os/CentOS-70-x86_64 --noautoconsole --extra-args "ks=http://192.168.0.2/os/kscfg/rdo03-ks.cfg console=tty0 console=ttyS0,115200n8 ksdevice=eth0 ip=192.168.0.23 netmask=255.255.255.0 gateway=192.168.0.1" --graphics vnc,listen=0.0.0.0,port=5903

4. インストール状況を確認する。
   (terminalを新規に開く)
   # virsh console rdo01
     ※ 最後まで進んだらEnterを押下
   (terminalを新規に開く)
   # virsh console rdo02
     ※ 最後まで進んだらEnterを押下
   (terminalを新規に開く)
   # virsh console rdo03
     ※ 最後まで進んだらEnterを押下

5. インストール後ゲストマシンを起動する
   # virsh list --all
   # virsh start rdo01
   # virsh start rdo02
   # virsh start rdo03
============================================================

============================================================
(ゲストマシンの設定 rdo01/rdo02/rdo03)
1. NetworkManagerを無効化
   (RHELの仮想化ドキュメントによるとNetworkManagerはbridgeに対応していないとのこと)
   # systemctl disable NetworkManager
   # systemctl enable network
   # systemctl stop NetworkManager 
   # systemctl start network
   # echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network
     ※ /etc/sysconfig/network-scripts/ifcfg-(eth名)の、「GATEWAY0」から、
        「0」を削除し、「GATEWAY」にしてもよい

2. /etc/resolv.confにドメインを追記
   # echo "domain func.org" >> /etc/resolv.conf

3. rpmを最新化する
   # yum update -y

4. ゲストマシンを停止する。
   # init 0

5. Hostマシンでスナップショットを取得
   (Hostマシン上で)
   # virsh snapshot-create-as rdo01 rdo01-install-snap "initial install and settings"
   # virsh snapshot-create-as rdo02 rdo02-install-snap "initial install and settings"
   # virsh snapshot-create-as rdo03 rdo03-install-snap "initial install and settings"

6. ゲストマシンの起動
   (Hostマシン上で)
   # virsh start rdo01
   # virsh start rdo02
   # virsh start rdo03
============================================================
