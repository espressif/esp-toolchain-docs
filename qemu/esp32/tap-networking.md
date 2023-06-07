# Tap Networking

This page outlines experimental setup when running ESP32 on QEMU with TAP network interface. This is a bit more complex than user mode, especially when running QEMU in Docker. We would have to:

* Start docker in network privileged mode
* Create a bridge to connect TAP interface to the default eth0 interface inside the container
* Additional network config on the docker hosting machine to connect the docker interface to global network

## Docker start
Start docker:

```
sudo docker run -it -v ~/qemu:/repos --rm  --cap-add=NET_ADMIN qemu /bin/bash
```

where `qemu` is the name of the image containing `qemu-system-xtensa`.

## Networking inside a container

```
ip link add br0 type bridge
mkdir /dev/net
mknod /dev/net/tun c 10 200
chmod 666 /dev/net/tun
ip tuntap add dev tap0 mode tap
ip link set tap0 master br0
ip link set eth0 master br0
ip link set tap0 up
ip link set br0 up
```

## Start QEMU in a container

```
qemu-system-xtensa \
  -nographic     \
  -machine esp32     \
  -drive file=mqtt.bin,if=mtd,format=raw \
  -nic tap,model=open_eth,ifname=tap0,downscript=no,script=no
```

## Host networking

* Configure it on the machine hosting docker
* Typically need to start dhcp-server on docker network interface (in case ESP32 works as Station/Ethernet node)
* Example of `ifconfig`:

```
example:
docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:d7ff:fe05:2668  prefixlen 64  scopeid 0x20<link>
```

* example of dhcp-d setup (on linux)

```
option domain-name "local";
default-lease-time 600;
max-lease-time 7200;
INTERFACES="docker0";
authoritative;

subnet 172.17.0.0 netmask 255.255.0.0 {
range 172.17.0.2 172.17.0.3;
option routers 172.17.0.1;
option domain-name-servers 8.8.8.8;
}
```