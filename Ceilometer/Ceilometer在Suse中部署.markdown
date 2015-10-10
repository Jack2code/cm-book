# Ceilometer在Suse中部署

标签（空格分隔）： Ceilometer

---
[TOC]

## 部署Ceilometer
进入Ceilometer根目录，`cd /usr/l00210728/ceilometer`

执行命令 `python setup.py install`


## 备忘
1. ceilometer安装前指标项配置文件为setup.cfg，在ceilometer根目录下；ceilometer安装后，指标项配置文件路径为python安装下，例如：/usr/lib64/python2.6/site-packages/ceilometer-2013.2.4.dev21.g27a67f4-py2.6.egg-info/entry_points.txt。ceilometer默认安装并不会采集所有支持的指标项，所以一般需要修改配置，配置时可参考源码进行配置。
2. 重启服务
    service openstack-ceilometer-agent-central restart
    service openstack-ceilometer-agent-compute restart
    service openstack-ceilometer-collector restart
    service openstack-ceilometer-api restart

    openstack-ceilometer-agent-central
    openstack-ceilometer-api

## 常用命令
1. 查看应用是否安装
    whereis XXX
2. 查看zypper源
    zypper repos
3. Python安装
    python setup.py install
    python stup.py develop（貌似这是开发模式）
4. 源码编译Python
    tar zxvf Python2.7.8.tar.gz
    cd Python2.7.8
    ./configure
    make 
    make install
    mv 
    ln -s 
5. 加载镜像
    sudo mount -o loop -t iso9660 /media/SLES-11-SP3-DVD-x86_64-GM-DVD1.iso /mnt
6. keystone用户列表
    keystone --debug user-list
7. keystone终端列表
    keystone endpoint-list
8. 加载keystone配置？
    source /root/openrc
9. ceilometer错误日志位置
    vi /var/log/ceilometer/api.log
10. Suse中Ceilometer安装位置 
    cd /usr/lib64/python2.6/site-packages/ceilometer/api/controllers
11. DB操作文件位置 
    vi /usr/lib64/python2.6/site-packages/ceilometer/storage/impl_mongodb.py
12. API文件位置
    vi /usr/lib64/python2.6/site-packages/ceilometer/api/controllers/v2.py
13. Ceilometer配置文件位置 
    vi /etc/ceilometer/ceilometer.conf
14. 去掉openstack认证
    修改ceilometer.conf文件里面的auth_strategy为noauth（默认是keystone)
15. ceilometer采集指标配置文件
    vi /usr/lib64/python2.6/site-packages/ceilometer-2013.2.4.dev21.g27a67f4-py2.6.egg-info/entry_points.txt
16. 查看虚机列表
    nova list

## 常见问题
1. ImportError: cannot import name HTTPSHandle
    yum安装openssl和openssl-devel。然后重新编译python。(http://daiqingyang.blog.51cto.com/1070509/1275432)




