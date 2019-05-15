## ceph 集群搭建 {#ceph-集群搭建}

#### 1.安装ceph-deploy {#1安装ceph-deploy}

* ceph-deploy：yum install ceph-deploy
* Ntp: yum install ntp ntpdate ntp-doc

#### 2.安装SSH服务器： {#2安装ssh服务器}

* 每个节点都安装ssh: yum install openssh-server

#### 3.创建ceph部署用户： {#3创建ceph部署用户}

* ssh user@ceph-server

```
     sudo useradd -d /home/{username} -m {username}
     sudo passwd {username}
```

#### 4.配置免密登陆: {#4配置免密登陆}

* 执行：ssh-keygen
* 将密钥复制到每个Ceph节点，替换{username}为您使用Create a Ceph Deploy User创建的用户名

```
    ssh-copy-id {username}@node1
    ssh-copy-id {username}@node2
    ssh-copy-id {username}@node3
```

* 修改~/.ssh/config 文件

```
        Host node1
        Hostname node1          
 User {username}
        Host node2
        Hostname node2          
 User {username}
        Host node3
        Hostname node3          
 User {username}

```

###### 将Ceph监视器节点的Ceph -mon服务和Ceph osd和MDSs的Ceph服务添加到公共区域，并确保设置是永久性的，以便在重新启动时启用它们 {#将ceph监视器节点的ceph-mon服务和ceph-osd和mdss的ceph服务添加到公共区域并确保设置是永久性的以便在重新启动时启用它们}

##### 监视器： {#监视器}

```
sudo firewall-cmd --zone=public --add-service=ceph-mon --permanent
```

##### OSD和MDS: {#osd和mds}

```
sudo firewall-cmd --zone=public --add-service=ceph --permanent
```

* 为iptables添加6789Ceph监视器的端口6800:7300 和Ceph OSD的端口:

```
sudo iptables -A INPUT -i esn33 -p tcp -s 192.168.147.129/255.255.255.0 --dport 6789 -j ACCEPT
>  /sbin/service iptables save(确保重启生效)
```

* #### 如果想要重新开始，请执行以下操作以清除Ceph软件包，并清除其所有数据和配置 {#如果想要重新开始请执行以下操作以清除ceph软件包并清除其所有数据和配置}

```
ceph-deploy purge {ceph-node} [{ceph-node}]  ------彻底删除ceph软件包
ceph-deploy purgedata {ceph-node} [{ceph-node}]  ------将保存ceph软件包
ceph-deploy forgetkeys
rm ceph.*

```

#### 开始部署： {#开始部署}

* 创建一个带有一个Ceph Monitor和三个Ceph OSD守护进程的Ceph存储集群，一旦群集达到状态，通过添加第四个Ceph OSD守护进程，元数据服务器和另外两个Ceph监视器来扩展它。
* 1.创建cluster目录：

```
 (1)mkdir my-cluster  --->cd my-cluster
```

* 2.创建集群：

```
(1)指定节点为主机名：
①ceph-deploy new node1
(2)安装ceph包：
①ceph-deploy install node1 node2 node3
(3)部署初始监控并收集密钥：
①ceph-deploy mon create-initial
完成此过程后，您的本地目录应具有以下密钥环：

        ceph.client.admin.keyring
        ceph.bootstrap-mgr.keyring
        ceph.bootstrap-osd.keyring
        ceph.bootstrap-mds.keyring
        ceph.bootstrap-rgw.keyring
        ceph.bootstrap-rbd.keyring
        ceph.bootstrap-rbd-mirror.keyring
(4)复制配置文件及密钥到管理节点和其他ceph节点：
①ceph-deploy admin node1 node2 node3
(5)部署管理器守护程序：
①ceph-deploy mgr create node1
(6)添加OSD,即假设每个节点都有未使用的磁盘
/dev/sdb    
①ceph-deploy osd create --data /dev/sdb node1    
②ceph-deploy osd create --data /dev/sdb node2    
③ceph-deploy osd create --data /dev/sdb node3
(7)检查集群运行状况：
①ssh node1 sudo ceph health    
②ssh node1 sudo ceph -s 
```

#### 扩展集群 {#扩展集群}

* 添加元数据服务器
  > ceph-deploy mds create node1
* 添加监视器
  > ceph-deploy mon add node2 node3
* 查看仲裁状态
  > ceph quorum\_status --format json-pretty
* 添加管理员：

```
Ceph Manager守护进程以活动/备用模式运行。部署其他管理器守护程序可确保在一个守护程序或主机发生故障时，另一个守护程序或主机可以在不中断服务的情况下接管
```

* 部署其他管理守护程序

```
>   ceph-deploy mgr create node2 node3
```

* 添加RGW实例

> 要使用Ceph的Ceph对象网关组件，必须部署RGW实例

```
    ceph-deploy rgw create node1-可以通过编辑ceph.conf
更改端口：
-       [client]
-       rgw frontends = civetweb port=80
```

* ### 块设备启动 {#块设备启动}

#### 1. 安装ceph {#1-安装ceph}

```
1.在admin节点上，用ceph-deploy 为节点ceph-client安装ceph    
> ceph-deploy install ceph-client
2.在admin节点上，用ceph-deploy复制Ceph的配置文件和ceph.client.admin.keyring到ceph-client
> ceph-deploy admin ceph-client
```

#### 2.创建块设备池 {#2创建块设备池}

```
1. 在ceph节点创建池
> ceph osd pool create mytest 8
2. 在ceph 节点，使用rbd工具初始化池以供RBD使用：
> rbd pool init mytest
```

#### 3.配置块设备 {#3配置块设备}

```
1
. 
在
ceph-client
节点上，创建块设备映像。
>
 sudo rbd create foo --size 
4096
 --image-feature layering -m 
192.168
.
147.129
 -k /etc/ceph/ceph
.client
.admin
.keyring
 -
p
 mytest

2
. 
在
ceph-client
节点上，将图像映射到块设备。
>
 sudo rbd map foo --name client
.admin
 -m 
192.168
.
147.129
 -k /etc/ceph/ceph
.client
.admin
.keyring
 -
p
 mytest

3
. 
通过在
ceph-client 
节点上创建文件系统来使用块设备。
>
 sudo mkfs
.ext4
 -m0 /dev/rbd/mytest/foo

4
. 
在
ceph-client
节点上挂载文件系统。
>
 sudo mkdir /mnt/ceph-block-device
    
>
 sudo mount /dev/rbd/mytest/foo /mnt/ceph-block-device
    
>
 cd /mnt/ceph-block-device

```

* ### cephFS {#cephfs}

```
1.
在
admin
节点上，用
ceph-deploy 
为节点
ceph-
client
安装
ceph
    
>
 ceph-deploy install ceph-
client
2.
在
admin
节点上，用
ceph-deploy
复制
Ceph
的配置文件和
ceph.
client
.admin.keyring
到
ceph-
client
>
 ceph-deploy admin ceph-
client
```

#### 1.创建文件系统 {#1创建文件系统}

```
-     ceph osd
 pool 
create cephfs_data 
<
pg_num
>

-     ceph osd
 pool 
create cephfs_metadata 
<
pg_num
>

-     ceph fs new 
<
fs_name
>
 cephfs_metadata cephfs_data

```

#### 2.创建秘密文件 {#2创建秘密文件}

```
1.
复制文件系统用户密钥
>
 cat ceph.client.admin.keyring
    
>
>
>
 [client.admin]
    
>
>
>
    key = AQCj2YpRiAe6CxAA7/
ETt7Hcl9IyxyYciVs47w
==  
2
，将
AQCj2YpRiAe6CxAA7/
ETt7Hcl9IyxyYciVs47w
== 
放入空文件（
admin.secret
）
```

#### 3.内核驱动程序 {#3内核驱动程序}

```
1
.  
将
cephFS
挂载为内核驱动程序

        sudo mkdir /mnt/mycephfs
        sudo mount -t ceph 
192.168
.147
.129
:
6789
:/ /mnt/mycephfs

2
.  Ceph
存储集群默认使用身份验证。指定用户
name 
和
secretfile
您在“
创建机密文件”部分中创建的用户

        sudo mount -t ceph 
192.168
.147
.129
:
6789
:/ /mnt/mycephfs -o name=admin,secretfile=admin.secret

```

#### 4.用户空间中的文件系统 {#4用户空间中的文件系统}

```
1. 
将
CephFS
挂载为用户空间（
FUSE
）中的文件系统

    sudo mkdir ~
/mycephfs

    sudo ceph-fuse -m {ip-address-of-monitor}
:6789
 ~
/mycephfs

2. Ceph
存储集群默认使用身份验证

    sudo ceph-fuse -k 
./ceph.client.admin.keyring
 -m 192.168.0.1
:6789
 ~
/mycephfs
```

* ### ceph对象网关 {#ceph对象网关}

```
1.
在
admin
节点上，用
ceph-deploy 
为节点
ceph-
client
安装
ceph
    
>
 ceph-deploy install ceph-
client
2.
在
admin
节点上，用
ceph-deploy
复制
Ceph
的配置文件和
ceph.
client
.admin.keyring
到
ceph-
client
>
 ceph-deploy admin ceph-
client
```

#### 1.安装ceph对象网关 {#1安装ceph对象网关}

```
1. 
如果使用
civetweb 
的默认端口，必须使用
firewalld 
或
 iptables
打开它
2. 
在
admin
节点上执行，安装
ceph
对象的网关包：
    ceph-deploy install --rgw cos2 cos3 cos4 ceph-client
```

#### 2.创建对象网关实例： {#2创建对象网关实例}

```
1. 在admin服务器上创建ceph对象网关的实例client-node
    ceph-deploy rgw create cos2

```

#### 3.配置ceph对象网关实例 {#3配置ceph对象网关实例}

```
1. 更改默认端口（例如，更改为端口80
），请修改
Ceph
配置文件。添加标题为“
 [client.rgw.
<
client-node
>
]
替换
<
client-node
>
为
Ceph
客户端节点的短节点名称”的部分（即）。例如，如果您的节点名称是
，请在以下部分之后添加如下所示的部分：
hostname -sclient-node[global]
    [client.rgw.client-node]
        rgw_frontends = 
"civetweb port=80"

2. 
要使新端口设置生效，重新启动
Ceph
对象网关，运行以下命令：

        sudo systemctl restart ceph-radosgw.service
   
在
Red Hat Enterprise Linux 6
和
Ubuntu
上，运行以下命令：

        sudo
 service 
radosgw restart 
id
=rgw.
<
short-hostname
>

3. 
检查以确保您选择的端口在节点的防火墙（例如，端口
80
）上是否打开。如果未打开，请添加端口并重新加载防火墙配置。例如：

        sudo firewall-cmd --list-all
        sudo firewall-cmd 
--zone
=public --add-port 80/tcp --permanent
        sudo firewall-cmd --reload

```

### \#\#\# 此时请求\#\#\# {#此时请求}

```
1. http://
<
client-node
>
:80

将得到：
<
?
xml version=
"1.0"
 encoding=
"UTF-8"
?
>
<
ListAllMyBucketsResultxmlns="http:
//
s3.amazonaws.com
/
do
c
/
2006-03-01
/"
>
<
Owner
>
<
ID
>
anonymous
<
/
ID
>
<
DisplayName
>
<
/
DisplayName
>
<
/
Owner
>
<
Buckets
>
<
/
Buckets
>
<
/
ListAllMyBucketsResult
>
```



