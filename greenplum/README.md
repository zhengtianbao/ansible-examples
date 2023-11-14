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
