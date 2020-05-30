# Install qemu-kvm libvirt

## Platform

- HW  
  「仮想化拡張機能」をサポートした CPU を使用していること。サポートされていれば `grep` の戻り値に整数が返る。
  ```bash
  $ grep -c -E '(vmx|svm)' /proc/cpuinfo
  6
  ```
- Distribution  
  本ドキュメントの確認環境に Ubuntu 20.04 を使用していますが、Ubuntu 18.04 でも手順は同じです。 
  ```bash
  $ lsb_release -id
  Distributor ID: Ubuntu
  Description:    Ubuntu 20.04 LTS
  ```

## Install qemu-kvm libvirt

- パッケージをインストール  
  以下のコマンドを実行しインストールします。
  ```bash
  $ sudo apt-get install --no-install-recommends \
  > qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils
  ```
- Recommend  
  強めのお勧めです。インストールされていないと不便に感じると思います。
  - virtinst: `virt-install` コマンド
  - virt-manager: GUI
  - qemu-utils: `qemu-img` コマンド

## Define

### Networks

#### Isolated network 【optional】

- Reference
  - [Isolated network config](https://libvirt.org/formatnetwork.html#examplesPrivate)
- Work Dir  
  任意の PATH に作業ディレクトリを作成します。
  ```bash
  $ mkdir -p ~/Work/config/libvirt
  ```
- 定義ファイルの作成
  ```bash
  $ vim ~/Work/config/libvirt/network-define_br_kvm0250.xml
  ~
  <network>
    <name>br_kvm0250</name>
    <bridge name='br_kvm0250'/>
    <mac address='52:54:00:1f:fa:fe'/>
    <ip address="172.31.250.254" netmask="255.255.255.0">
      <dhcp>
        <range start='172.31.250.50' end='172.31.250.99'/>
      </dhcp>
    </ip>
  </network>
  ~
  :wq
  ```
- ネットワークの作成
  ```bash
  $ virsh net-define --file ~/Work/config/libvirt/network-define_br_kvm0250.xml
  Network br_kvm0250 defined from /home/<dev-user>/Work/config/libvirt/network-define_br_kvm0250.xml
  ```
- 自動起動を有効化
  ```bash
  $ virsh net-autostart --network br_kvm0250
  Network br_kvm0250 marked as autostarted
  ```
- ネットワークの起動
  ```bash
  $ virsh net-start --network br_kvm0250
  ```
- Check  
  Bridge が作成された事を確認します。
  ```bash
  $ ip addr show br_kvm0250
  38: br_kvm0250: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default qlen 1000
      link/ether 52:54:00:1f:fa:fe brd ff:ff:ff:ff:ff:ff
      inet 172.31.250.254/24 brd 172.31.250.255 scope global br_kvm0250
         valid_lft forever preferred_lft forever
  ```
- NAPT  
  BackEnd Network としてではなく上流(Internet 等)に抜けたい場合は `virbr0` と同様に NAPT させる必要があります。
  以下に任意の Dest Port のみを NAPT させる設定をサンプルとして記述します。
  ```bash
  $ IPT='/sbin/iptables'
  $ UPSTREAM_IF='br_mgmt0'
  $ KVM_NAPT_IF='br_kvm0250'
  $ KVM_NAPT_PFIX='172.31.250.0/24'
  
  $ sudo ${IPT} --table nat --append POSTROUTING --out-interface ${UPSTREAM_IF} --source ${KVM_NAPT_PFIX} \
  > --match conntrack --ctstate NEW --match udp --protocol udp --dport  53 --jump MASQUERADE
  $ sudo ${IPT} --table nat --append POSTROUTING --out-interface ${UPSTREAM_IF} --source ${KVM_NAPT_PFIX} \
  > --match conntrack --ctstate NEW --match tcp --protocol tcp --dport  53 --jump MASQUERADE
  $ sudo ${IPT} --table nat --append POSTROUTING --out-interface ${UPSTREAM_IF} --source ${KVM_NAPT_PFIX} \
  > --match conntrack --ctstate NEW --match udp --protocol udp --dport 123 --jump MASQUERADE
  $ sudo ${IPT} --table nat --append POSTROUTING --out-interface ${UPSTREAM_IF} --source ${KVM_NAPT_PFIX} \
  > --match conntrack --ctstate NEW --match tcp --protocol tcp --match multiport --dports   80,443 --jump MASQUERADE
  $ sudo ${IPT} --table nat --append POSTROUTING --out-interface ${UPSTREAM_IF} --source ${KVM_NAPT_PFIX} \
  > --match conntrack --ctstate NEW --match tcp --protocol tcp --match multiport --dports 3128,8080 --jump MASQUERADE
  ```
  ```bash
  <vm-host>:~$ sudo ip route add default via 172.31.250.254
  <vm-host>:~$ curl -x http://<ProxyHost>:<ProxyPort>/ www.goo.ne.jp
  <vm-host>:~$ dig @<NameServer> www.goo.ne.jp
  ```
  ```bash
  $ sudo iptables -nvL POSTROUTING --line-number --table nat
  Chain POSTROUTING (policy ACCEPT 80 packets, 7023 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  1        0     0 MASQUERADE  all  --  *      !docker0  172.17.0.0/16        0.0.0.0/0           
  2     181K   15M LIBVIRT_PRT  all  --  *      *       0.0.0.0/0            0.0.0.0/0           
  3       24  2467 MASQUERADE  all  --  *      docker_gwbridge  0.0.0.0/0            0.0.0.0/0            ADDRTYPE match src-type LOCAL
  4        0     0 MASQUERADE  all  --  *      !docker_gwbridge  172.18.0.0/16        0.0.0.0/0           
  5        1    82 MASQUERADE  udp  --  *      br_mgmt0  172.31.250.0/24      0.0.0.0/0            ctstate NEW udp dpt:53
  6        0     0 MASQUERADE  tcp  --  *      br_mgmt0  172.31.250.0/24      0.0.0.0/0            ctstate NEW tcp dpt:53
  7        0     0 MASQUERADE  udp  --  *      br_mgmt0  172.31.250.0/24      0.0.0.0/0            ctstate NEW udp dpt:123
  8        0     0 MASQUERADE  tcp  --  *      br_mgmt0  172.31.250.0/24      0.0.0.0/0            ctstate NEW tcp multiport dports 80,443
  9        1    60 MASQUERADE  tcp  --  *      br_mgmt0  172.31.250.0/24      0.0.0.0/0            ctstate NEW tcp multiport dports 3128,8080
  ```
