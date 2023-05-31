

sudo dd bs=1M if=Downloads/2022-09-22-raspios-bullseye-arm64-lite.img of=/dev/disk2

or raspberry pi imager

RPi cmdline.txt (https://elinux.org/RPi_cmdline.txt)


```
cgroup_memory=1 cgroup_enable=memory ip=192.168.1.10::192.168.1.254:255.255.255.0:masterpi:eth0:off
cgroup_memory=1 cgroup_enable=memory ip=192.168.1.11::192.168.1.254:255.255.255.0:node1pi:eth0:off
cgroup_memory=1 cgroup_enable=memory ip=192.168.1.12::192.168.1.254:255.255.255.0:node2pi:eth0:off
cgroup_memory=1 cgroup_enable=memory ip=192.168.1.13::192.168.1.254:255.255.255.0:node3pi:eth0:off




curl -sfL https://get.k3s.io | K3S_URL=https://masterpi:6443 K3S_TOKEN=K10ba582a323fb25d9f87fbdc856eb8027daf92acbdbe7751ecfed902aaae7a29f5::server:8d0a814b44efe20372d2ae1f3719193e sh -
```

RPi config.txt

arm_64bit=1   (not needed on x64 version)

touch ssh
