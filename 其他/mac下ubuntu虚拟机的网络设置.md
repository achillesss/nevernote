# mac 下 ubuntu 虚拟机的网络设置

## 需求：

* 无局域网时，虚拟机和主机能相互访问
* 有局域网时，虚拟机能通过主机网络访问外网
* 有局域网时，外网其他机器可以访问虚拟机
* 虚拟机对主机的ip固定

---

## 网络模式

一般情况下有三种：NAT地址转换，Host-only，Bridge

### NAT:

让虚拟机借助NAT（网络地址转换）功能，通过主机所在的网络来访问外网。

此模式中NAT的网络是一个虚拟的网络，主机局域网中其他的主机无法访问虚拟机，仅主机可以访问虚拟机；同时虚拟机可以访问主机局域网中其他主机

### Host-only:

此模式下，虚拟网络是一个全封闭的网络，只能让主机和虚拟机相互访问。虚拟机不能通过主机网络访问外网

### Bridge:

此模式下，本地无力网卡和虚拟网卡通过虚拟交换机进行桥接，两网卡处于同一网段。此时虚拟机可以和局域网中任一主机相互访问，同时可以访问外网。虚拟机有自己独立的ip

---

## 设置：

1. 同时添加上面三种网络
2. 启动虚拟机
3. 修改配置：

```bash
sudo vim /etc/network/interfaces
```

```
auto lo

iface lo inet loopback


# The primary network interface

auto eth0

iface eth0 inet dhcp


# Virtualbox Host-only mode

auto eth1

iface eth1 inet static

address 192.168.56.101

netmask 255.255.255.0


#Virtualbox Bridged mode

auto eth2

iface eth2 inet static

address 192.168.1.121

netmask 255.255.255.0

gateway 192.168.1.1
```

其中 eth0／eth1／eth2的名字先要通过

```
ifconfig
```

来确认。

最后

```
sudo /etc/init.d/networking restart
```