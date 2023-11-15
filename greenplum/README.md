## 安装步骤

本文记录在UnionTech OS Server 20 (2010e)环境下安装greenplum 6.25.3的步骤。

### 节点说明

| IP              | hostname  | role            | 数据盘  |
| --------------  | --------- | --------------  | ------ |
| 192.168.122.59  | mdw       | master, ansible | vda    |
| 192.168.122.133 | smdw      | standby_master  | vda    |
| 192.168.122.31  | sdw1      | segment1        | vda    |

数据盘必须是挂载块存储，ansible中会进行xfs格式化

### 所有节点操作

编辑/etc/hosts 增加域名解析

```
192.168.122.59 mdw
192.168.122.133 smdw
192.168.122.31 sdw1
```

安装依赖

```
# yum install apr apr-util bash bzip2 curl krb5 libcurl libevent libxml2 libyaml zlib openldap openssh openssl openssl-libs perl readline rsync R sed tar zip xfsdump xfsprogs-devel
```

### ansible节点预处理

修改hostname

```
[root@mdw ~]# hostnamectl set-hostname mdw
```

修改用户权限

```
[root@mdw ~]# echo "RemoveIPC=no" >> /etc/systemd/logind.conf
[root@mdw ~]# systemctl restart systemd-logind.service 
# visudo 取消行注释 %wheel        ALL=(ALL)       NOPASSWD: ALL
[root@mdw ~]# visudo
```

安装ansible以及依赖

```
[root@mdw ~]# yum install ansible python3-dnf
```

修改ansible模块代码以支持UOS

```
vi /usr/lib/python3.7/site-packages/ansible/modules/system/hostname.py
```

在642行 CentOSHostname附近增加Uos支持

```
class UosHostname(Hostname):
     platform = 'Linux'
     distribution = 'Uos'
     strategy_class = RedHatStrategy
```

### ansible配置

inventory 配置文件

```
[greenplum_master]
mdw

[greenplum_standby]
smdw

[greenplum_segments]
sdw1

[greenplums]
mdw
smdw
sdw1
```

site.yml

```yaml
- name: deploy greenplum
  hosts: greenplums
  remote_user: root
  roles:
    - greenplum
```

测试ansible节点联通性

```
[root@mdw ~]# export ANSIBLE_HOST_KEY_CHECKING=False
[root@mdw ~]# export ANSIBLE_PYTHON_INTERPRETER=/usr/bin/python3
[root@mdw ~]# ansible -i inventory all -m ping
```

执行ansible-playbook

```
[root@mdw ~]# git clone https://github.com/zhengtianbao/ansible-examples.git ansible-examples
[root@mdw ~]# cd ansible-examples/greenplum
[root@mdw ~]# ansible-playbook -i inventory site.yml 
```

第一遍执行后因为节点重启会断开连接
注释 tasks/main.yml 中 `- meta: flush_handlers` 行，再次执行

```
[root@mdw ~]# export ANSIBLE_HOST_KEY_CHECKING=False
[root@mdw ~]# export ANSIBLE_PYTHON_INTERPRETER=/usr/bin/python3
[root@mdw ~]# ansible-playbook -i inventory site.yml 
```

至此 ansible预部署完毕，后续需要手动执行操作

### greenplum master节点操作

将gpadmin用户增加wheel组

```
[root@mdw ~]# usermod -aG wheel gpadmin
```

增加软链接 否则gpinitsystem命令执行报错

```
[root@mdw ~]# ln -s /usr/lib64/libreadline.so.8 /usr/lib64/libreadline.so.7
```

以gpadmin用户执行以下步骤

```
[root@mdw ~]# su - gpadmin
[gpadmin@mdw ~]$ source /usr/local/greenplum-db/greenplum_path.sh
[gpadmin@mdw ~]$ ssh-keygen -t rsa
[gpadmin@mdw ~]$ ssh-copy-id smdw
[gpadmin@mdw ~]$ ssh-copy-id sdw1
[gpadmin@mdw ~]$ gpssh-exkeys -f /home/gpadmin/all_nodes
[STEP 1 of 5] create local ID and authorize on local host
  ... /home/gpadmin/.ssh/id_rsa file exists ... key generation skipped

[STEP 2 of 5] keyscan all hosts and update known_hosts file

[STEP 3 of 5] retrieving credentials from remote hosts
  ... send to smdw
  ... send to sdw1

[STEP 4 of 5] determine common authentication file content

[STEP 5 of 5] copy authentication files to all remote hosts
  ... finished key exchange with smdw
  ... finished key exchange with sdw1

[INFO] completed successfully
```

检查节点联通性

```
[gpadmin@mdw ~]$ gpssh -f /home/gpadmin/all_nodes -e "ls -l /usr/local/greenplum-db"
```

安装

```
[gpadmin@mdw ~]$ gpinitsystem -c /home/gpadmin/gpinitsystem_config -s smdw
```

在master和standby节点 添加访问权限

```
[gpadmin@mdw ~]$ echo "host     all         gpadmin         0.0.0.0/0       md5" >> /data1/gpmaster/gpseg-1/pg_hba.conf
```

重新加载gpdb配置文件

```
[gpadmin@mdw ~]$ gpstop -u
```

查看状态

```
[gpadmin@mdw ~]$ gpstate -s
```

设置gpadmin访问密码 创建jfbrother库

```
[gpadmin@mdw ~]$ psql postgres gpadmin
psql (9.4.26)
Type "help" for help.

postgres=# alter user gpadmin encrypted password 'gpadmin';
ALTER ROLE
postgres=# create database jfbrother owner gpadmin;
postgres=# \q
```

至此，greenplum集群搭建完毕，可以通过 `jdbc:postgresql://192.168.122.59:5432/jfbrother` 用户名`gpadmin` 密码`gpadmin` 连接到数据库

## 集群运行中增加segment节点

### 节点说明

| IP              | hostname  | role            | 数据盘  |
| --------------  | --------- | --------------  | ------ |
| 192.168.122.59  | mdw       | master, ansible | vda    |
| 192.168.122.133 | smdw      | standby_master  | vda    |
| 192.168.122.31  | sdw1      | segment1        | vda    |
| 192.168.122.104 | sdw2      | segment2        | vda    |

新增节点 192.168.122.104

### 所有节点操作

修改/etc/hosts 添加域名解析

```
192.168.122.59 mdw
192.168.122.133 smdw
192.168.122.31 sdw1
192.168.122.104 sdw2
```

### 新增节点操作

安装依赖

```
# yum install apr apr-util bash bzip2 curl krb5 libcurl libevent libxml2 libyaml zlib openldap openssh openssl openssl-libs perl readline rsync R sed tar zip xfsdump xfsprogs-devel
```

### ansible节点配置修改

inventory 配置文件

```
[greenplum_master]
[greenplum_standby]

[greenplum_segments]
sdw2

[greenplums]
sdw2
```

site.yml

```yaml
- name: deploy greenplum
  hosts: greenplums
  remote_user: root
  roles:
    - greenplum
```

测试ansible节点联通性

```
[root@mdw ~]# export ANSIBLE_HOST_KEY_CHECKING=False
[root@mdw ~]# export ANSIBLE_PYTHON_INTERPRETER=/usr/bin/python3
[root@mdw ~]# ansible -i inventory all -m ping
```

执行ansible-playbook

```
[root@mdw ~]# git clone https://github.com/zhengtianbao/ansible-examples.git ansible-examples
[root@mdw ~]# cd ansible-examples/greenplum
[root@mdw ~]# ansible-playbook -i inventory site.yml 
```

至此 初始化sdw2环境完毕，后续需要手动执行操作

### 在master节点操作

新增节点列表

```
[gpadmin@mdw ~]# cat << EOF > /home/gpadmin/newseg_nodes
sdw2
EOF
```

配置ssh key

```
[gpadmin@mdw ~]$ source /usr/local/greenplum-db/greenplum_path.sh
[gpadmin@mdw ~]$ ssh-copy-id -i .ssh/id_rsa.pub gpadmin@sdw2
[gpadmin@mdw ~]$ gpssh-exkeys -e /home/gpadmin/all_nodes -x /home/gpadmin/newseg_nodes
```

查看现有集群的配置规则

```
[gpadmin@mdw ~]$ psql postgres gpadmin
psql (9.4.26)
Type "help" for help.

postgres=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port | hostname | address |         datadir         
------+---------+------+----------------+------+--------+------+----------+---------+-------------------------
    1 |      -1 | p    | p              | n    | u      | 5432 | mdw      | mdw     | /data1/gpmaster/gpseg-1
    2 |       0 | p    | p              | n    | u      | 6000 | sdw1     | sdw1    | /data1/gpdatap1/gpseg0
    3 |       1 | p    | p              | n    | u      | 6001 | sdw1     | sdw1    | /data1/gpdatap2/gpseg1
    4 |      -1 | m    | m              | s    | u      | 5432 | smdw     | smdw    | /data1/gpmaster/gpseg-1
(4 rows)

postgres=# 
```

而gpexpand文件每行为：

```
hostname|address|port|datadir|dbid|content|preferred_role

hostname			主机名
address				类似主机名
port				segment监听端口
datadir			segment data目录,注意是全路径
dbid				gp集群的唯一ID，可以到gp_segment_configuration中获得，必须顺序累加
content				可以到gp_segment_configuration中获得，必须顺序累加
prefered_role		角色(p或m)(primary , mirror)
```

创建gpexpend配置文件

```
[gpadmin@mdw ~]$ cat << EOF > gpexpand_inputfile
sdw2|sdw2|6000|/data1/gpdatap1/gpseg2|5|2|p
sdw2|sdw2|6001|/data1/gpdatap2/gpseg3|6|3|p
EOF
```
这里sdw2的数据可以通过sdw1的数据推测出来，参考下文最终结果

执行部署

```
[gpadmin@mdw ~]$ gpexpand -i gpexpand_inputfile
```

查看状态 

```
[gpadmin@mdw ~]$ gpstate
[gpadmin@mdw ~]$ gpstate -x
```

```
[gpadmin@mdw ~]$ psql postgres gpadmin
psql (9.4.26)
Type "help" for help.

postgres=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port | hostname | address |         datadir         
------+---------+------+----------------+------+--------+------+----------+---------+-------------------------
    1 |      -1 | p    | p              | n    | u      | 5432 | mdw      | mdw     | /data1/gpmaster/gpseg-1
    2 |       0 | p    | p              | n    | u      | 6000 | sdw1     | sdw1    | /data1/gpdatap1/gpseg0
    3 |       1 | p    | p              | n    | u      | 6001 | sdw1     | sdw1    | /data1/gpdatap2/gpseg1
    4 |      -1 | m    | m              | s    | u      | 5432 | smdw     | smdw    | /data1/gpmaster/gpseg-1
    5 |       2 | p    | p              | n    | u      | 6000 | sdw2     | sdw2    | /data1/gpdatap1/gpseg2
    6 |       3 | p    | p              | n    | u      | 6001 | sdw2     | sdw2    | /data1/gpdatap2/gpseg3
(6 rows)

postgres=# 

```
