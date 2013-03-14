# OpenStack folsom 在CentOS6.3上的配置

## 安装NTP服务

### 安装NTP

    # yum install -y ntp 

### 修改 */etc/ntp.conf* ，限定为中国地区的ntp服务器，具体serveri信息参考（[China — cn.pool.ntp.org]）

### 启动ntp服务

    # service ntpd start  
    # chkconfig ntpd on

PS: 理想做法是控制节点与某个ntp服务器同步时间，计算节点向控制节点同步时间

## 安装MySQL

### 安装MySQL
    # yum install mysql mysql-server MySQL-python  
PS: 安装过程中需要设置root账户密码，建议设为Ops146

### 设置mysql服务为开机启动  
    # chkconfig --level 2345 mysqld on  
    # service mysqld start  
    
**注意！如果安装过程中未提示输入root账户密码，则安装完毕后root账户没有密码，此时需要人工设置密码:**

    mysql> USE mysql;  
    mysql> UPDATE user SET Password=password('Ops146') WHERE User='root';  
    mysql> FLUSH PRIVILEGES; 
    mysql> \q
    
## 安装RabbitMQ
### 安装RabbitMQ
    # yum install openstack-utils memcached
    # yum install -y rabbitmq-server
    # /etc/init.d/rabbitmq-server start
PS: 另外也可以选择qpid，如果这么做的话需要修改  nova.conf：rpc_backend=nova.rpc.impl_qpid

## 安装并配置身份认证服务——keystone   

### 安装keystone   
    # yum install openstack-utils openstack-keystone python-keystoneclient   

### 创建keystone库并授权   
    # mysql -uroot -p     
    mysql> CREATE DATABASE keystone;
    mysql> GRANT ALL ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone';   
    mysql> GRANT ALL ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone';   
    mysql> GRANT ALL ON keystone.* TO 'keystone'@'Ops146' IDENTIFIED BY 'keystone';
    mysql> \q   
注意：上面3条grant指令最好都写上，否则也许会出错，这里假设控制节点的hostname是Ops146
PS：openstack对数据库的初始化操作有专门的指令，例如：
    # openstack-db --init --service keystone
但这条指令不太熟悉，如果这样写的话都是默认值，不方便修改，所以还是直接操纵MySQL方便

### 删除sqlite，指向MySQL   
现在版本的OpenStack已经默认指向MySQL了，所以这一步省略

### 修改 */etc/keystone/keystone.conf*
    connection = mysql://keystone:keystone@162.105.133.146/keystone   
PS：这里假设控制节点的IP是162.105.133.146，口令须与之前数据库授权时的口令相同，这里使用了默认值

### 设置service token（一般是取一个随机字符串）
    # export ADMIN_TOKEN=$(openssl rand -hex 10)   
    # openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $ADMIN_TOKEN

### 重启keystone服务，使上述配置文件生效
    # service openstack-keystone start && chkconfig openstack-keystone on   

### 初始化keystone库
    # keystone-manage db_sync 

### 创建各种tenants，users，roles   
此步涉及大量重复工作，如果人工执行命令很不人道，所以使用官方脚本：

PS：脚本中所有endpoint地址都是127.0.0.1，此外用户名和密码也是默认的，建议修改后再执行
8. troubleshooting   
- 随便执行几个keystone命令，比如:
     keystone --os-username=admin --os-password=Ops146 --os-auth-url=http://162.105.133.146:35357/v2.0 token-get   
- 查看/var/log/keystone/路径下的日志文件（如果没有开启debug选项的话此时应该是空）
- 如果日志有异常可以在命令中增加-debug选项查看debug信息   
- 为便于运行keystone命令，可以创建一个keystonerc文件，具体内容：
     export OS_USERNAME=admin   
     export OS_PASSWORD=Ops146   
     export OS_TENANT_NAME=demo   
     export OS_AUTH_URL=http://162.105.133.146:35357/v2.0



