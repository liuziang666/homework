1）主机网卡配置
2）关闭防火墙、selinux及libvirtd服务

[root@qll251 ~]# systemctl stop firewalld
[root@qll251 ~]# systemctl disable firewalld

[root@qll251 ~]# vim /etc/selinux/config
改：SELINUX=enforcing
为：SELINUX=disabled

[root@qll251 ~]# systemctl stop libvirtd.service
[root@qll251 ~]# systemctl disable libvirtd.service

[root@qll251 ~]# reboot #重启生效

3）安装epel源

yum -y install epel-release
4）CentOS 部分常用软件安装

yum install -y vim net-tools  bash-completion-extras git

5）配置主机名及hosts文件

[root@qll251 ~]# hostname qll251
[root@qll251 ~]# echo "qll251" > /etc/hostname
[root@qll251 ~]# echo "192.168.1.251  qll251" >> /etc/hosts

6）同步时间

[root@qll251 ~]# yum -y install ntp
[root@qll251 ~]# systemctl start ntpd
[root@qll251 ~]# systemctl enable ntpd

7）配置 pip 镜像源，方便快速下载python库

[root@qll251 ~]# mkdir ~/.pip
[root@qll251 ~]# vim ~/.pip/pip.conf
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host=mirrors.aliyun.com

3.2 安装基础包和docker服务

1）安装基础包

yum -y install python-devel libffi-devel gcc openssl-devel  python-pip

2）升级pip版本，不然后期安装会有报警

3）安装docker-ce

安装依赖包
yum -y install yum-utils device-mapper-persistent-data lvm2

添加docker-ce yum源文件
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
1
安装docker-ce
 yum -y install docker-ce

启动docker服务
systemctl start docker
systemctl enable docker

4）指定docker 镜像加速器

[root@qll251 ~]# vim /etc/docker/daemon.json
        添加如下内容：
{
  "registry-mirrors": ["https://0i6rnnzu.mirror.aliyuncs.com"]
}


5）设置docker volume卷挂载方式

[root@qll251 ~]# mkdir /etc/systemd/system/docker.service.d
[root@qll251 ~]# vim  /etc/systemd/system/docker.service.d/kolla.conf
  # 添加如下内容
[Service]
MountFlags=shared

解释一下：MountFlags=shared，表示当宿主机新增分区时，docker服务无需重启即可识别。添加此参数后期OpenStack中使用cinder存储时，新加磁盘也比较方便
6）重启使配置生效

systemctl daemon-reload
systemctl restart docker
systemctl enable docker

3.3 从github 获取Kolla和Kolla-Ansible

1）安装ansible

yum -y install ansible
1
2）下载kolla及kolla-ansible代码

git clone https://github.com/openstack/kolla -b stable/stein
git clone https://github.com/openstack/kolla-ansible -b stable/stein
  # 如果已有镜像，只执行第二步即可

3）手动安装kolla-ansible

python ~/kolla-ansible/setup.py install
1
4）安装kolla-ansible需要依赖包


[root@qll251 ~]# pip install -r /root/kolla-ansible/requirements.txt
1
2


如果出现此报错，我们强制更新即可；

执行：

[root@qll251 ~]# pip install --ignore-installed PyYAML
1


5）安装kolla需要依赖包

[root@qll251 ~]# pip install -r /root/kolla/requirements.txt
1
注意：如果出现类似如下错误：

requests 2.20.0 has requirement idna<2.8,>=2.5, but you'll have idna 2.4 which is incompatible

同样，强制更新requets库即可；

[root@qll251 ~]# pip install --ignore-installed requests
1
6）拷贝配置文件

[root@qll251 ~]# cd ~/kolla-ansible/
[root@qll251 kolla-ansible]# cp -r ./etc/kolla/* /etc/kolla/
[root@qll251 kolla-ansible]# cp ./ansible/inventory/* /etc/kolla/

#看下我们都拷贝了哪些文件
[root@qll251 ~]# ls /etc/kolla/
all-in-one  globals.yml  multinode  passwords.yml
[root@qll251 ~]#

配置文件解释：

all-in-one #安装单节点OpenStack的ansible自动安装配置文件
multinode # 安装多节点OpenStack的ansible自动安装配置文件
globals.yml # 部署OpenStack的自定义配置文件
passwords.yml #存放OpenStack各个服务的密码

6）生成随机密码

[root@qll251 ~]# kolla-genpwd
1
使用kolla提供的密码生成工具自动生成OpenStack各服务的密码，如果密码不填充，后面的部署环境检查时不会通过的。
7）修改随机密码文件

# 为了方便登录Dashboard，我们将密码修改为123123
[root@qll251 ~]# vim /etc/kolla/passwords.yml
 165 keystone_admin_password: 123123
8）修改globals.yml配置文件

[root@qll251 ~]#  vim /etc/kolla/globals.yml
# 指定镜像的系统版本
 15 kolla_base_distro: "centos"
# 指定安装方式
 18 kolla_install_type: "binary"
# 指定安装stein版本的OpenStack
 21 openstack_release: "stein"
# 本次实验采用all-in-one模式，未启用高可用。填写宿主机IP即可
 31 kolla_internal_vip_address: "192.168.1.251"
# OpenStack内部管理网络
 89 network_interface: "eth0"
# Neutron外网网络
107 neutron_external_interface: "eth1"
# 本次实验采用all-in-one模式，未启用高可用
192 enable_haproxy: "no"

1）生成SSH Key，并授信本节点

ssh-keygen
ssh-copy-id root@192.168.1.251
2）配置单节点all-in-one配置文件

[root@qll251 ~]# vim /etc/kolla/all-in-one
# 将文件中所有的localhost替换成qll251
:1,$s/localhost/qll251/

# 去掉文件中所有包含“ansible_connection=local”
:1,$s/ansible_connection=local//

3）带有kolla的引导服务器部署依赖关系

[root@qll251 ~]# kolla-ansible -i /etc/kolla/all-in-one bootstrap-servers

4）对主机执行预部署检查

[root@qll251 ~]# kolla-ansible -i /etc/kolla/all-in-one prechecks


5）拉取OpenStack镜像

[root@qll251 ~]# kolla-ansible -i /etc/kolla/all-in-one  pull


6）执行OpenStack部署

kolla-ansible -i /etc/kolla/all-in-one  deploy
1
7）验证部署

kolla-ansible -i /etc/kolla/all-in-one  post-deploy
1



同时也生成了admin用户的凭证， 即/etc/kolla/admin-openrc.sh文件

我们看下该凭证：



4 登录OpenStack云平台

在浏览器中输入：http://192.168.1.251

用户名：admin

密码：123123


到此已完成OpenStack云平台的部署，明天我们再来讨论下OpenStack 云平台基本使用方法及利用OpenStack客户端命令创建一台测试云主机。

