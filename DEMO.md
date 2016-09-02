# DEMO

[つくって学ぶLinuxコンテナの裏側 // Speaker Deck](https://speakerdeck.com/hayajo/tukututexue-bulinuxkontenafalseli-ce)

## 準備

```bash
$ git clone https://github.com/ailispaw/yapc-barge.git
$ cd yapc-barge
$ vagrant up
$ vagrant ssh
[bargee@barge ~]$ 
```

以下、`$ ` がローカルマシン、`[bargee@barge ~]$ ` がホスト側プロンプトで、それ以外はコンテナ内。

## デモ

### Namespace デモ (p.20)

```bash
[bargee@barge ~]$ sudo /vagrant/yapc.1 /bin/bash
[root@barge bargee]# ps auxf
PID   USER     COMMAND
    1 root     /bin/bash
    2 root     ps auxf
[root@barge bargee]# hostname yapc; hostname
yapc
[root@barge bargee]# exit
exit
[bargee@barge ~]$ hostname
barge
```

- ✔ PIDが独立していることを確認
- ✔ ホスト名変更がホストへ影響がないことを確認

### cgroup デモ (p.28)

```bash
[bargee@barge ~]$ sudo YAPC_CPU_QUOTA=50000 /vagrant/yapc.2 /bin/bash -c "yes >/dev/null"
```

#### もう一つ別のターミナルを開いてホストに入る
```bash
$ vagrant ssh
[bargee@barge ~]$ top
Mem: 598000K used, 425232K free, 58100K shrd, 21068K buff, 480032K cached
CPU: 50.0% usr  0.1% sys  0.0% nic 49.8% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 0.22 0.11 0.05 2/93 592
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
  581   574 root     R     8916  0.8   0 49.7 yes
```

- ✔ ホストでtopを実行し、上記コンテナプロセスのCPU使用率が50%付近になることを確認

#### 終了後、Ctrl-C でコンテナを終了。

### Capability デモ (p.39)

```bash
[bargee@barge ~]$ sudo /vagrant/yapc.3 ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
ping: permission denied (are you root?)
```

- ✔ pingができないことを確認

```bash
[bargee@barge ~]$ sudo YAPC_CAPS="cap_net_raw" /vagrant/yapc.3 ping 127.0.0.1
PING 127.0.0.1 (127.0.0.1): 56 data bytes
64 bytes from 127.0.0.1: seq=0 ttl=64 time=0.044 ms
64 bytes from 127.0.0.1: seq=1 ttl=64 time=0.092 ms
```

- ✔ pingができることを確認

### pivot_root, overlayfs デモ（１） (p.48)

```bash
[bargee@barge ~]$ sudo /vagrant/yapc.4 /bin/bash
[root@barge /]# touch /yapc; ls -l /yapc
-rw-r--r--    1 root     root             0 Jul 28 22:10 /yapc
```

#### もう一つ別のターミナルを開いてホストに入る
```bash
$ vagrant ssh
[bargee@barge ~]$ ls -l /yapc
ls: /yapc: No such file or directory
[bargee@barge ~]$ sudo -i
[root@barge ~]# ls -l /tmp/yapc.*/upper/yapc
-rw-r--r--    1 root     root             0 Jul 28 22:10 /tmp/yapc.4.635.XXvECcd5/upper/yapc
```

- ✔ ホストで/yapcが存在しないことを確認
- ✔ ホストで/tmp/yapc-<PID>.XXXXXX/upper/yapcが存在することを確認

### pivot_root, overlayfs デモ（２） (p.49)

```bash
[bargee@barge ~]$ sudo YAPC_ROOT=centos /vagrant/yapc.4 /bin/bash
[root@barge /]# cat /etc/os-release
NAME="CentOS Linux"
VERSION="7 (Core)"
ID="centos"
ID_LIKE="rhel fedora"
VERSION_ID="7"
PRETTY_NAME="CentOS Linux 7 (Core)"
ANSI_COLOR="0;31"
CPE_NAME="cpe:/o:centos:centos:7"
HOME_URL="https://www.centos.org/"
BUG_REPORT_URL="https://bugs.centos.org/"

CENTOS_MANTISBT_PROJECT="CentOS-7"
CENTOS_MANTISBT_PROJECT_VERSION="7"
REDHAT_SUPPORT_PRODUCT="centos"
REDHAT_SUPPORT_PRODUCT_VERSION="7"

[root@barge /]# yum install -y epel-release; yum install -y sl; sl
```

- ✔ SL が走ることを確認

### Network Namespaceとveth デモ（１） (p.56)

```bash
[bargee@barge ~]$ sudo YAPC_NET=1 YAPC_CAPS="cap_net_raw,cap_net_admin" /vagrant/yapc.a ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
6: eth0@if7: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN mode DEFAULT group default qlen 1000
    link/ether 7e:ed:dc:72:e1:f1 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

- ✔ コンテナ内で独立したネットワークインターフェースが見えることを確認

### Network Namespaceとveth デモ（２〜４） (p.57~)

#### ホスト側でブリッジを作成する
```bash
[bargee@barge ~]$ sudo ip link add name yapc0 type bridge
[bargee@barge ~]$ sudo ip link set dev yapc0 up
[bargee@barge ~]$ sudo ip a add 10.0.0.1/24 broadcast 10.0.0.255 label yapc0 dev yapc0
[bargee@barge ~]$ ip link | grep yapc0
8: yapc0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
```

#### ネットワークネームスペースを有効にしたコンテナでbashを実行
```bash
[bargee@barge ~]$ sudo YAPC_NET=1 YAPC_CAPS="cap_net_raw,cap_net_admin" /vagrant/yapc.a /bin/bash
[root@barge /]# 
```

#### もう一つ別のターミナルを開いて、ホスト側のvethをブリッジに登録する
```bash
$ vagrant ssh
[bargee@barge ~]$ ip link | grep veth
12: vethyapc1038@if11: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
[bargee@barge ~]$ sudo ip link set dev vethyapc1038 up
[bargee@barge ~]$ sudo ip link set dev vethyapc1038 master yapc0
[bargee@barge ~]$ ip link | grep veth
12: vethyapc1038@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master yapc0 state UP mode DEFAULT group default qlen 1000
```

#### コンテナ側のvethにIPアドレスを割り当る
```bash
[root@barge /]# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 3a:77:b0:f6:ed:1a brd ff:ff:ff:ff:ff:ff link-netnsid 0
[root@barge /]# ip link set dev eth0 up
[root@barge /]# ip a add 10.0.0.10/24 dev eth0
[root@barge /]# ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
11: eth0@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 3a:77:b0:f6:ed:1a brd ff:ff:ff:ff:ff:ff link-netnsid 0
[root@barge /]# ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1): 56 data bytes
64 bytes from 10.0.0.1: seq=0 ttl=64 time=0.204 ms
64 bytes from 10.0.0.1: seq=1 ttl=64 time=0.073 ms
```

- ✔ ホストにpingができることを確認
