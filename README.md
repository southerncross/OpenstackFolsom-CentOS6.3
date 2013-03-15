# OpenStack folsom 在CentOS6.3上的配置

## 安装说明

### 配置环境

    - 操作系统CentOS6.3
    - IP地址162.105.133.146
    - 所有操作都是在root账户下进行

### 安装流程

    1. 准备工作
    2. 安装NTP、rabbitmq等第三方服务
    3. 安装keystone
    4. 安装glance
    5. 安装nova
    6. 安装dashboard
    7. 测试——运行一个实例
    
PS：实际安装步骤会有些许差别

## 具体流程

### 配置软件源

导入testing源

    # wget http://mirror.neu.edu.cn/fedora/epel/6/i386/epel-release-6-8.noarch.rpm
    # rpm -i epel-release-6-8.noarch.rpm
    
修改 */etc/yum.repos.d/epel-testing.repo* 文件，将其中的enable全部改为1

### 安装NTP服务

安装NTP

    # yum install -y ntp 
    
配置修改 */etc/ntp.conf* ，限定为中国地区的ntp服务器，具体serveri信息参考（[China — cn.pool.ntp.org]）

启动ntp服务

    # service ntpd start  
    # chkconfig ntpd on
    
PS: 理想做法是控制节点与某个ntp服务器同步时间，计算节点向控制节点同步时间

### 安装MySQL

    # yum install mysql mysql-server MySQL-python  
    
PS: 安装过程中需要设置root账户密码，建议设为Ops146

设置mysql服务为开机启动  

    # chkconfig --level 2345 mysqld on  
    # service mysqld start  
    
**注意！如果安装过程中未提示输入root账户密码，则安装完毕后root账户没有密码，此时需要人工设置密码:**

    mysql> USE mysql;  
    mysql> UPDATE user SET Password=password('Ops146') WHERE User='root';  
    mysql> FLUSH PRIVILEGES; 
    mysql> \q
    
### 安装RabbitMQ和cache服务

    # yum install openstack-utils memcached
    # yum install -y rabbitmq-server
    # /etc/init.d/rabbitmq-server start
    
PS: 另外也可以选择qpid，如果这么做的话需要修改  nova.conf：rpc_backend=nova.rpc.impl_qpid

### 安装并配置身份认证服务——keystone   

    # yum install openstack-utils openstack-keystone python-keystoneclient   

创建keystone库并授权   

    # mysql -uroot -p     
    mysql> CREATE DATABASE keystone;
    mysql> GRANT ALL ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone';   
    mysql> GRANT ALL ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'keystone';   
    mysql> GRANT ALL ON keystone.* TO 'keystone'@'Ops146' IDENTIFIED BY 'keystone';
    mysql> \q   
    
PS：上面3条grant指令最好都写上，否则也许会出错，这里假设控制节点的hostname是Ops146

PS：openstack对数据库的初始化操作有专门的指令，例如：

    # openstack-db --init --service keystone
    
但这条指令不太熟悉，如果这样写的话都是默认值，不方便修改，所以还是直接操纵MySQL方便

修改 */etc/keystone/keystone.conf*

    connection = mysql://keystone:keystone@162.105.133.146/keystone   
    
PS：这里假设控制节点的IP是162.105.133.146，口令须与之前数据库授权时的口令相同，这里使用了默认值

设置service token（一般是取一个随机字符串）

    # export ADMIN_TOKEN=$(openssl rand -hex 10)   
    # openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token $ADMIN_TOKEN

重启keystone服务，使上述配置文件生效

    # service openstack-keystone start && chkconfig openstack-keystone on   

初始化keystone库

    # keystone-manage db_sync 

创建各种tenants，users，roles   
此步涉及大量重复工作，如果人工执行命令很不人道，所以使用官方脚本[sample_data.sh]：

    #!/usr/bin/env bash
    
    # Copyright 2013 OpenStack LLC
    #
    # Licensed under the Apache License, Version 2.0 (the "License"); you may
    # not use this file except in compliance with the License. You may obtain
    # a copy of the License at
    #
    #      http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS, WITHOUT
    # WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the
    # License for the specific language governing permissions and limitations
    # under the License.

    # Sample initial data for Keystone using python-keystoneclient
    #
    # This script is based on the original DevStack keystone_data.sh script.
    #
    # It demonstrates how to bootstrap Keystone with an administrative user
    # using the SERVICE_TOKEN and SERVICE_ENDPOINT environment variables
    # and the administrative API.  It will get the admin_token (SERVICE_TOKEN)
    # and admin_port from keystone.conf if available.
    #
    # Disable creation of endpoints by setting DISABLE_ENDPOINTS environment variable.
    # Use this with the Catalog Templated backend.
    #
    # A EC2-compatible credential is created for the admin user and
    # placed in etc/ec2rc.
    #
    # Tenant               User      Roles
    # -------------------------------------------------------
    # demo                 admin     admin
    # service              glance    admin
    # service              nova      admin
    # service              ec2       admin
    # service              swift     admin

    CONTROLLER_PUBLIC_ADDRESS=${CONTROLLER_PUBLIC_ADDRESS:-localhost}
    CONTROLLER_ADMIN_ADDRESS=${CONTROLLER_ADMIN_ADDRESS:-localhost}
    CONTROLLER_INTERNAL_ADDRESS=${CONTROLLER_INTERNAL_ADDRESS:-localhost}

    TOOLS_DIR=$(cd $(dirname "$0") && pwd)
    KEYSTONE_CONF=${KEYSTONE_CONF:-/etc/keystone/keystone.conf}
    if [[ -r "$KEYSTONE_CONF" ]]; then
        EC2RC="$(dirname "$KEYSTONE_CONF")/ec2rc"
    elif [[ -r "$TOOLS_DIR/../etc/keystone.conf" ]]; then
        # assume git checkout
        KEYSTONE_CONF="$TOOLS_DIR/../etc/keystone.conf"
        EC2RC="$TOOLS_DIR/../etc/ec2rc"
    else
        KEYSTONE_CONF=""
        EC2RC="ec2rc"
    fi

    # Extract some info from Keystone's configuration file
    if [[ -r "$KEYSTONE_CONF" ]]; then
        CONFIG_SERVICE_TOKEN=$(sed 's/[[:space:]]//g' $KEYSTONE_CONF | grep ^admin_token= | cut -d'=' -f2)
        CONFIG_ADMIN_PORT=$(sed 's/[[:space:]]//g' $KEYSTONE_CONF | grep ^admin_port= | cut -d'=' -f2)
    fi

    export SERVICE_TOKEN=${SERVICE_TOKEN:-$CONFIG_SERVICE_TOKEN}
    if [[ -z "$SERVICE_TOKEN" ]]; then
        echo "No service token found."
        echo "Set SERVICE_TOKEN manually from keystone.conf admin_token."
        exit 1
    fi

    export SERVICE_ENDPOINT=${SERVICE_ENDPOINT:-http://$CONTROLLER_PUBLIC_ADDRESS:${CONFIG_ADMIN_PORT:-35357}/v2.0}

    function get_id () {
        echo `"$@" | grep ' id ' | awk '{print $4}'`
    }

    #
    # Default tenant
    #
    DEMO_TENANT=$(get_id keystone tenant-create --name=demo \
                                                --description "Default Tenant")

    ADMIN_USER=$(get_id keystone user-create --name=admin \
                                            --pass=secrete)

    ADMIN_ROLE=$(get_id keystone role-create --name=admin)

    keystone user-role-add --user-id $ADMIN_USER \
                           --role-id $ADMIN_ROLE \
                           --tenant-id $DEMO_TENANT
    
    #
    # Service tenant
    #
    SERVICE_TENANT=$(get_id keystone tenant-create --name=service \
                                                   --description "Service Tenant")
    
    GLANCE_USER=$(get_id keystone user-create --name=glance \
                                              --pass=glance)
    
    keystone user-role-add --user-id $GLANCE_USER \
                           --role-id $ADMIN_ROLE \
                           --tenant-id $SERVICE_TENANT
    
    NOVA_USER=$(get_id keystone user-create --name=nova \
                                            --pass=nova \
                                            --tenant-id $SERVICE_TENANT)
    
    keystone user-role-add --user-id $NOVA_USER \
                           --role-id $ADMIN_ROLE \
                           --tenant-id $SERVICE_TENANT
    
    EC2_USER=$(get_id keystone user-create --name=ec2 \
                                           --pass=ec2 \
                                           --tenant-id $SERVICE_TENANT)
    
    keystone user-role-add --user-id $EC2_USER \
                           --role-id $ADMIN_ROLE \
                           --tenant-id $SERVICE_TENANT
    
    SWIFT_USER=$(get_id keystone user-create --name=swift \
                                             --pass=swiftpass \
                                             --tenant-id $SERVICE_TENANT)
    
    keystone user-role-add --user-id $SWIFT_USER \
                           --role-id $ADMIN_ROLE \
                           --tenant-id $SERVICE_TENANT
    
    #
    # Keystone service
    #
    KEYSTONE_SERVICE=$(get_id \
    keystone service-create --name=keystone \
                            --type=identity \
                            --description="Keystone Identity Service")
    if [[ -z "$DISABLE_ENDPOINTS" ]]; then
        keystone endpoint-create --region RegionOne --service-id $KEYSTONE_SERVICE \
            --publicurl "http://$CONTROLLER_PUBLIC_ADDRESS:\$(public_port)s/v2.0" \
            --adminurl "http://$CONTROLLER_ADMIN_ADDRESS:\$(admin_port)s/v2.0" \
            --internalurl "http://$CONTROLLER_INTERNAL_ADDRESS:\$(public_port)s/v2.0"
    fi
    
    #
    # Nova service
    #
    NOVA_SERVICE=$(get_id \
    keystone service-create --name=nova \
                            --type=compute \
                            --description="Nova Compute Service")
    if [[ -z "$DISABLE_ENDPOINTS" ]]; then
        keystone endpoint-create --region RegionOne --service-id $NOVA_SERVICE \
            --publicurl "http://$CONTROLLER_PUBLIC_ADDRESS:\$(compute_port)s/v1.1/\$(tenant_id)s" \
            --adminurl "http://$CONTROLLER_ADMIN_ADDRESS:\$(compute_port)s/v1.1/\$(tenant_id)s" \
            --internalurl "http://$CONTROLLER_INTERNAL_ADDRESS:\$(compute_port)s/v1.1/\$(tenant_id)s"
    fi
    
    #
    # Volume service
    #
    VOLUME_SERVICE=$(get_id \
    keystone service-create --name=volume \
                            --type=volume \
                            --description="Nova Volume Service")
    if [[ -z "$DISABLE_ENDPOINTS" ]]; then
        keystone endpoint-create --region RegionOne --service-id $VOLUME_SERVICE \
            --publicurl "http://$CONTROLLER_PUBLIC_ADDRESS:8776/v1/\$(tenant_id)s" \
            --adminurl "http://$CONTROLLER_ADMIN_ADDRESS:8776/v1/\$(tenant_id)s" \
            --internalurl "http://$CONTROLLER_INTERNAL_ADDRESS:8776/v1/\$(tenant_id)s"
    fi
    
    #
    # Image service
    #
    GLANCE_SERVICE=$(get_id \
    keystone service-create --name=glance \
                            --type=image \
                            --description="Glance Image Service")
    if [[ -z "$DISABLE_ENDPOINTS" ]]; then
        keystone endpoint-create --region RegionOne --service-id $GLANCE_SERVICE \
            --publicurl "http://$CONTROLLER_PUBLIC_ADDRESS:9292" \
            --adminurl "http://$CONTROLLER_ADMIN_ADDRESS:9292" \
            --internalurl "http://$CONTROLLER_INTERNAL_ADDRESS:9292"
    fi
    
    #
    # EC2 service
    #
    EC2_SERVICE=$(get_id \
    keystone service-create --name=ec2 \
                            --type=ec2 \
                            --description="EC2 Compatibility Layer")
    if [[ -z "$DISABLE_ENDPOINTS" ]]; then
        keystone endpoint-create --region RegionOne --service-id $EC2_SERVICE \
            --publicurl "http://$CONTROLLER_PUBLIC_ADDRESS:8773/services/Cloud" \
            --adminurl "http://$CONTROLLER_ADMIN_ADDRESS:8773/services/Admin" \
            --internalurl "http://$CONTROLLER_INTERNAL_ADDRESS:8773/services/Cloud"
    fi
    
    #
    # Swift service
    #
    SWIFT_SERVICE=$(get_id \
    keystone service-create --name=swift \
                            --type="object-store" \
                            --description="Swift Service")
    if [[ -z "$DISABLE_ENDPOINTS" ]]; then
        keystone endpoint-create --region RegionOne --service-id $SWIFT_SERVICE \
            --publicurl   "http://$CONTROLLER_PUBLIC_ADDRESS:8888/v1/AUTH_\$(tenant_id)s" \
            --adminurl    "http://$CONTROLLER_ADMIN_ADDRESS:8888/v1" \
            --internalurl "http://$CONTROLLER_INTERNAL_ADDRESS:8888/v1/AUTH_\$(tenant_id)s"
    fi
    
    # create ec2 creds and parse the secret and access key returned
    RESULT=$(keystone ec2-credentials-create --tenant-id=$SERVICE_TENANT --user-id=$ADMIN_USER)
    ADMIN_ACCESS=`echo "$RESULT" | grep access | awk '{print $4}'`
    ADMIN_SECRET=`echo "$RESULT" | grep secret | awk '{print $4}'`
    
    # write the secret and access to ec2rc
    cat > $EC2RC <<EOF
    ADMIN_ACCESS=$ADMIN_ACCESS
    ADMIN_SECRET=$ADMIN_SECRET
    EOF
    
PS：脚本中所有endpoint地址都是127.0.0.1，此外用户名和密码也是默认的，建议修改后再执行

troubleshooting   

    - 随便执行几个keystone命令，比如: keystone --os-username=admin --os-password=Ops146 --os-auth-url=http://162.105.133.146:35357/v2.0 token-get   
    - 查看/var/log/keystone/路径下的日志文件（如果没有开启debug选项的话此时应该是空）
    - 如果日志有异常可以在命令中增加-debug选项查看debug信息   

为便于运行keystone命令，可以创建一个keystonerc文件一次性导入所需环境变量，内容如下：

    export OS_USERNAME=admin   
    export OS_PASSWORD=Ops146   
    export OS_TENANT_NAME=demo   
    export OS_AUTH_URL=http://162.105.133.146:35357/v2.0

### 安装和配置ImageService

安装image服务glance

    # yum install openstack-nova openstack-glance   
    # rm /var/lib/glange/glance.sqlite

创建数据库

     # mysql -uroot -p
     mysql> CREATE DATABSE glance;   
     mysql> GRANT ALL ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance';   
     mysql> GRANT ALL ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance';   
     mysql> GRANT ALL ON glance.* TO 'glance'@'Ops146' IDENTIFIED BY 'glance';
     mysql> \q   

编辑glance配置文件和paste.ini中间件文件：     

更新 */etc/glance/glance-api-paste.ini* ，修改[filter:authtoken]下的admin_*变量：   

     [filter:authtoken]   
     admin_tenant_name = service   
     admin_user = glance   
     admin_password = glance   
     
在 */etc/glance/glance-api.conf* 最后添加下述内容： 

     [keystone_authtoken]   
     auth_host = 127.0.0.1   
     auth_port = 35357   
     auth_protocol = http   
     admin_tenant_name = service   
     admin_user = glance   
     admin_password = glance   
  
     [paste_deploy]   
     # Name of the paste configuration file that defines the available pipelines config_file = /etc/glance/glance-api-paste.ini
   
     # Partial name of a pipeline in your paste configuration file with the  
     # service name removed. For example, if your paste section name is   
     # [pipeline:glance-api-keystone], you would configure the flavor below   
     # as 'keystone'.   
     flavor = keystone  
     
确保 */etc/glance/glance-api.conf* 指向MySQL而不是sqlite   

     sql_connection = mysql://glance:glance@162.105.133.146/glance   
     
更新 */etc/glance/glance-registry.conf* 最后一段，通过设置flavor=keystone来启用认证服务   

     [keystone_authtoken]   
     auth_host = 127.0.0.1   
     auth_port = 35357   
     auth_protocol = http   
     admin_tenant_name = service   
     admin_user = glance   
     admin_password = glance   
   
     [paste_deploy]   
     # Name of the paste configuration file that defines the available pipelines config_file = /etc/glance/glance-registry-paste.ini   
        
     # Partial name of a pipeline in your paste configuration file with the   
     #service name removed. For example, if your paste section name is   
     # [pipeline:glance-api-keystone], your would configure the flavor below 
     # as 'keystone'.   
     flavor = keystone  
     
更新 */etc/glance/glance-registry-paste.ini* 

     # Use this pipeline for keystone auth   
     [pipeline:glance-registry-keystone]   
     pipeline = authtoken contex registryapp   
     
确保 */etc/glance/glance-registry.conf* 指向的是MySQL而不是sqlite：

     sql_connection = mysql://glance:glance@162.105.133.146/glance 
     
重启glance服务使配置生效

    # service openstack-glance-registry restart  
    
初始化glance库

    # glance-manage db_sync   
     
重启glance-registry和glance-api服务

    # service openstack-glance-registry restart   
    # service openstack-glance-api restart   

Troubleshooting   

    - 查看/var/log/glance路径下的各种log文件,确保没有ERROR  
    - 随意执行几个glance命令，例如：glance image-list

#### 进一步验证image服务是否正常 

下载一个测试image

    # mkdir /root/images   
    # cd /root/images/   
    # wget http://smoser.brickies.net/ubuntu/ttylinux-uec/ttylinux-uec-amd64-12.1_2.6.35-22_1.tar.gz   
    # tar -zxvf ttylinux-uec-amd64-12.1_2.6.35-22_1.tar.gz   
    
创建openrc文件，便于执行命令：（注：这里的密码必须和sample_data.sh中创建的一致）

    export OS_USERNAME=admin   
    export OS_TENANT_NAME=demo   
    export OS_PASSWORD=Ops146   
    export OS_AUTH_URL=http://162.105.133.146:5000/v2.0/   
    export OS_REGION_NAME=RegionOne   
     
加载kernel  

    # glance image-create --name="tty-linux-kernel" --is-public true --disk-format=aki --container-format=aki < ttylinux-uec-amd64-12.1_2.6.35-22_1-vmlinuz，下面的命令都默认采用这种精简后的指令
    
PS：需要记住命令执行后返回的id，后面要用到
    
加载initrd   

    # glance image-create --name="tty-linux-ramdisk" --is-public true --disk-format=ari --container-format=ari < ttylinux-uec-amd64-12.1_2.6.35-22_1-loader   
    
PS：同样要记住命令执行后返回的id，后面要用到

加载image

    # glance image-create --name="tty-linux" --is-public true --disk-format=ami --property kernel_id=cb77fbcf-89f4-441f-9ffb-47f3d045f445 --property ramdisk_id=e17e50b0-b7b3-474b-ac94-a861853bdb9b < ttylinux-uec-amd64-12.1_2.6.35-22_1.img 
    
这里填入之前2步的id

验证   

    # glance image-list  
    
这里应该会出现刚才加载的3个img，并且都是ACTIVE状态  

PS：这里官方文档上的教程有误，其文档中在执行image-create的时候没有指定--is-public，同时也没有指定owner，所以创建完毕后用glance image-list是看不到的，因为image没有owner而且是private的。（从官方文档的截图来看，虽然其没有指定--is-public但是是有owner的，但是根据我的实验，使用官方文档中的命令create的image的owner是NULL）这里的image-create命令已经做的修改，增加了--is-public true选项。


### 配置管理器

管理器分为KVM和Xen-based，KVM运行在libvirt上，Xen运行在XenAPI上  

KVM是Compute服务默认的管理器（这里暂时只考虑KVM，其他的先不考虑）  

为了打开KVM支持，需要在/etc/nova/nova.conf中添加如下配置项 （后面提供的nova.conf配置文件已包含，这里不用配置）  

     compute_driver=libvirt.LibvirtDriver   
     libvirt_type=kvm   
     
检查硬件是否支持虚拟化（如果grep有结果则支持）

    # egrep '(vmx|svm)' --color=always /proc/cpuinfo   
     
检查是否加载KVM模块（如果输出中有kvm则说明已经加载） 

    # lsmod | grep kvm
     
安装libvirt的一些东西，如果不执行则libvirt无法启动

    # yum -y install avahi
    # service messagebus start
    # service avahi-daemon start
    # service libvirtd start
    
PS：官方配置文档缺少这部分内容


### 配置网络

将网卡设置为混杂模式，这样就能接收到虚拟机发送的数据包了

    # ifconfig em2 promisc  
    
PS：这里网卡名是em2

创建网桥

创建文件 */etc/sysconfig/network-scrips/ifcfg-br100*

    DEVICE=br100
    TYPE=Bridge
    ONBOOT=yes
    DELAY=0
    BOOTPROTO=static
    IPADDR=192.168.100.1
    NETMASK=255.255.255.0
    
安装网桥工具

    # yum install bridge-utils
    
建立网桥（官网上提到了，一定要先建立网桥）

    # brctl addbr br100
    
重启使配置生效

    # /etc/init.d/network restart
    
PS：这个命令官方文档写错了


### 安装compute服务Nova

将selinux设置为许可模式

    # setenforce permissive
    
安装dnsmasq工具

    # yum install dnsmasq-utils

配置控制节点的SQL数据库

    # mysql -uroot -p
    mysql> CREATE DATABASE nova;
    mysql> GRANT ALL ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova';
    mysql> GRANT ALL ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
    mysql> GRANT ALL ON nova.* TO 'nova'@'Ops146' IDENTIFIED BY 'nova';
    mysql> \q

#### 安装nova

    # yum -y install openstack-nova
    
编辑 *nova.conf*   

PS: 这个nova.conf样例文件是在官方的样例上修改而来，可以直接用

    [DEFAULT]
    
    # LOGS/STATE
    verbose=True
    logdir=/var/log/nova
    state_path=/var/lib/nova
    lock_path=/var/lock/nova
    rootwrap_config=/etc/nova/rootwrap.conf
    
    # SCHEDULER
    compute_scheduler_driver=nova.scheduler.filter_scheduler.FilterScheduler
    
    # VOLUMES
    volume_driver=nova.volume.driver.ISCSIDriver
    volume_group=nova-volumes
    volume_name_template=volume-%s
    iscsi_helper=tgtadm
    
    # DATABASE
    sql_connection=mysql://nova:nova@162.105.133.146/nova
    
    # COMPUTE
    libvirt_type=kvm
    compute_driver=libvirt.LibvirtDriver
    instance_name_template=instance-%08x
    api_paste_config=/etc/nova/api-paste.ini

    # COMPUTE/APIS: if you have separate configs for separate services
    # this flag is required for both nova-api and nova-compute
    allow_resize_to_same_host=True
    
    # APIS
    osapi_compute_extension=nova.api.openstack.compute.contrib.standard_extensions
    ec2_dmz_host=162.105.133.146
    s3_host=162.105.133.146
    
    # RABBITMQ
    rabbit_host=162.105.133.146
    rpc_backend = nova.rpc.impl_kombu
    rabbit_max_retries=3
    rabbit_port=5672
    rabbit_retry_backoff=5
    rabbit_retry_interval=3

    # GLANCE
    image_service=nova.image.glance.GlanceImageService
    glance_api_servers=162.105.133.146:9292
    
    # NETWORK
    network_manager=nova.network.manager.FlatDHCPManager
    force_dhcp_release=True
    dhcpbridge = /usr/bin/nova-dhcpbridge
    dhcpbridge_flagfile=/etc/nova/nova.conf
    firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
    # Change my_ip to match each host
    my_ip=162.105.133.146
    public_interface=em2
    vlan_interface=em2
    flat_network_bridge=br100
    flat_interface=em2
    fixed_range=192.168.100.0/24

    # NOVNC CONSOLE
    novncproxy_base_url=http://162.105.133.146:6080/vnc_auto.html
    # Change vncserver_proxyclient_address and vncserver_listen to match each compute host
    vnc_enabled=true
    vncserver_proxyclient_address=162.105.133.146
    vnc_keymap=en-us
    vncserver_listen=162.105.133.146
    vncserver_proxyclient_address=162.105.133.146
    
    # AUTHENTICATION
    auth_strategy=keystone
    libvirt_inject_partition = -1
    [keystone_authtoken]
    auth_host = 127.0.0.1
    auth_port = 35357
    auth_protocol = http
    admin_tenant_name = service
    admin_user = nova
    admin_password = nova
    signing_dirname = /tmp/keystone-signing-nova

PS：注意其中的lock_path，不知道为什么，如果这么启动了，network会报错，说permission denied，所以这里要手动创建lock_path并修改属主：

    # mkdir /var/lock/nova
    # chown -R nova:nova /var/lock/nova
    
停止nova相关服务，否则初始化数据库的时候可能会有error

    # for svc in api objectstore compute network volume scheduler cert consoleauth console; do service openstack-nova-$svc stop; chkconfig openstack-nova-$svc on; done
    
初始化数据库（注意这里没有_）

     nova-manage db sync
     
此时会有一条debug信息，暂时不知道有什么影响

    2013-03-11 14:07:27 19585 DEBUG nova.utils [-] backend <module 'nova.db.sqlalchemy.migration' from '/usr/lib/python2.6/site-packages/nova/db/sqlalchemy/migration.pyc'> __get_backend /usr/lib/python2.6/site-packages/nova/utils.py:502
    
重启所有nova相关服务  

    # for svc in api objectstore compute network volume scheduler cert consoleauth console; do service openstack-nova-$svc restart; chkconfig openstack-nova-$svc on; done
    
执行这一步的时候也许会报下面几种错误    

    - rabbitmq问题，官网的配置文件号称是在配置rabbitmq，但实际上配置文件里写的是qpid的，同时如果按照qpid来装又有很多问题，最后干脆装rabbitmq！（本配置文件已解决）
    - compute问题，有关libvirt的，需要另外安装几个东西（本配置文件已解决）
    - network问题，permission denied /var/lock/nova，用chown -R /var/lock/nova即可，如果没有这个路径则要手工创建（本配置文件已解决）
    - volume问题，这个是因为vg名字不是nova-volumes造成的（本配置文件已解决）
    
验证

    # nova-manage service list
    
确保所有的service都是笑脸

    #service --status-all
    
确保除meta之外的openstack相关服务都是running状态

### 配置虚拟网络

    # nova-manage network create public --fixed_range_v4=192.168.100.0/24 --num_networks=1 --network_size=256 --bridge=br100
    
PS：貌似bridge_interface和bridge这两个选项的含义不同，因此官方配置文档上这条命令有误

执行这句会有一个debug信息

    2013-03-11 16:56:37 DEBUG nova.utils [req-ac6bd88d-846c-4c35-9b33-af9b79254508 None None] backend <module 'nova.db.sqlalchemy.api' from '/usr/lib/python2.6/site-packages/nova/db/sqlalchemy/api.pyc'> __get_backend /usr/lib/python2.6/site-packages/nova/utils.py:502
    
暂时不知道会有什么影响

### 配置novnc

    # yum install -y memcached mod-wgsi openstack-nova-novncproxy
    
PS：其余就是修改nova.conf文件，之前的nova.conf已经修改过了

    /etc/init.d/openstack-nova-consoleauth start
    /etc/init.d/openstack-nova-novncproxy start


### 安装dashboard

修改/etc/openstack-dashboard/local_settings

    import os
    
    from django.utils.translation import ugettext_lazy as _
    
    DEBUG = False
    TEMPLATE_DEBUG = DEBUG
    # TODO
    USE_SSL = False
    
    # Set SSL proxy settings:
    # For Django 1.4+ pass this header from the proxy after terminating the SSL,
    # and don't forget to strip it from the client's request.
    # For more information see:
    # https://docs.djangoproject.com/en/1.4/ref/settings/#secure-proxy-ssl-header
    # SECURE_PROXY_SSL_HEADER = ('HTTP_X_FORWARDED_PROTOCOL', 'https')

    # Specify a regular expression to validate user passwords.
    # HORIZON_CONFIG = {
    #     "password_validator": {
    #         "regex": '.*',
    #         "help_text": _("Your password does not meet the requirements.")
    #     },
    #    'help_url': "http://docs.openstack.org"
    # }
    
    LOCAL_PATH = os.path.dirname(os.path.abspath(__file__))

    # TODO
    # We need to change this to mysql, instead of sqlite
    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': 'horizon',
            'USER': 'horizon',
            'PASSWORD': 'horizon',
            'HOST': '162.105.133.146',
            'PORT': '3306',
        },
    }
    
    # Set custom secret key:
    # You can either set it to a specific value or you can let horizion generate a
    # default secret key that is unique on this machine, e.i. regardless of the
    # amount of Python WSGI workers (if used behind Apache+mod_wsgi): However, there
    # may be situations where you would want to set this explicitly, e.g. when
    # multiple dashboard instances are distributed on different machines (usually
    # behind a load-balancer). Either you have to make sure that a session gets all
    # requests routed to the same dashboard instance or you set the same SECRET_KEY
    # for all of them.
    # from horizon.utils import secret_key
    # SECRET_KEY = secret_key.generate_or_read_from_file(os.path.join(LOCAL_PATH, '.secret_key_store'))
    
    # We recommend you use memcached for development; otherwise after every reload
    # of the django development server, you will have to login again. To use
    # memcached set CACHE_BACKED to something like 'memcached://127.0.0.1:11211/'
    CACHE_BACKEND = 'memcached://127.0.0.1:11211/'

    # Send email to the console by default
    EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
    # Or send them to /dev/null
    #EMAIL_BACKEND = 'django.core.mail.backends.dummy.EmailBackend'
    
    # Configure these for your outgoing email host
    # EMAIL_HOST = 'smtp.my-company.com'
    # EMAIL_PORT = 25
    # EMAIL_HOST_USER = 'djangomail'
    # EMAIL_HOST_PASSWORD = 'top-secret!'
    
    # For multiple regions uncomment this configuration, and add (endpoint, title).
    # AVAILABLE_REGIONS = [
    #     ('http://cluster1.example.com:5000/v2.0', 'cluster1'),
    #     ('http://cluster2.example.com:5000/v2.0', 'cluster2'),
    # ]
    
    # TODO
    OPENSTACK_HOST = "162.105.133.146"
    OPENSTACK_KEYSTONE_URL = "http://%s:5000/v2.0" % OPENSTACK_HOST
    OPENSTACK_KEYSTONE_DEFAULT_ROLE = "Member"
    
    # Disable SSL certificate checks (useful for self-signed certificates):
    # OPENSTACK_SSL_NO_VERIFY = True
    
    # The OPENSTACK_KEYSTONE_BACKEND settings can be used to identify the
    # capabilities of the auth backend for Keystone.
    # If Keystone has been configured to use LDAP as the auth backend then set
    # can_edit_user to False and name to 'ldap'.
    #
    # TODO(tres): Remove these once Keystone has an API to identify auth backend.
    OPENSTACK_KEYSTONE_BACKEND = {
        'name': 'native',
        'can_edit_user': True
    }

    OPENSTACK_HYPERVISOR_FEATURES = {
        'can_set_mount_point': True
    }
    
    # OPENSTACK_ENDPOINT_TYPE specifies the endpoint type to use for the endpoints
    # in the Keystone service catalog. Use this setting when Horizon is running
    # external to the OpenStack environment. The default is 'internalURL'.
    #OPENSTACK_ENDPOINT_TYPE = "publicURL"
    
    # The number of objects (Swift containers/objects or images) to display
    # on a single page before providing a paging element (a "more" link)
    # to paginate results.
    API_RESULT_LIMIT = 1000
    API_RESULT_PAGE_SIZE = 20
    
    # The timezone of the server. This should correspond with the timezone
    # of your entire OpenStack installation, and hopefully be in UTC.
    TIME_ZONE = "UTC"
    
    LOGGING = {
            'version': 1,
            # When set to True this will disable all logging except
            # for loggers specified in this configuration dictionary. Note that
            # if nothing is specified here and disable_existing_loggers is True,
            # django.db.backends will still log unless it is disabled explicitly.
            'disable_existing_loggers': False,
            'handlers': {
                'null': {
                    'level': 'DEBUG',
                    'class': 'django.utils.log.NullHandler',
                },
                'console': {
                    # Set the level to "DEBUG" for verbose output logging.
                'level': 'INFO',
                    'class': 'logging.StreamHandler',
                },
             },
            'loggers': {
            # Logging from django.db.backends is VERY verbose, send to null
            # by default.
                'django.db.backends': {
                    'handlers': ['null'],
                    'propagate': False,
                },
                'horizon': {
                    'handlers': ['console'],
                    'propagate': False,
                },
                'openstack_dashboard': {
                    'handlers': ['console'],
                    'propagate': False,
                },
                'novaclient': {
                    'handlers': ['console'],
                    'propagate': False,
                },
                'keystoneclient': {
                    'handlers': ['console'],
                    'propagate': False,
                },
                'glanceclient': {
                    'handlers': ['console'],
                    'propagate': False,
                },
                'nose.plugins.manager': {
                    'handlers': ['console'],
                    'propagate': False,
                }
            }
    }

创建数据库

    # mysql -uroot -p
    mysql> create database horizon;
    mysql> grant all privileges on horizon.* to horizon@'%' identified by 'horizon';
    mysql> grant all privileges on horizon.* to horizon@'localhost' identified by 'horizon';
    mysql> grant all privileges on horizon.* to horizon@'Ops146' identified by 'horizon';
    mysql> flush privileges;
    mysql> \q
    
初始化表

    # /usr/share/openstack-dashboard/manage.py syncdb
    
启动http服务，此时还要对防火墙做一些设置，这里为了方便就直接关掉了

    # service iptables off
    # /etc/init.d/httpd start
    
### 在controller节点上启动一个实例来检测功能是否正常

在nova中添加secgroup，开放ssh和icmp

    # nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
    # nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
    
添加mykey密钥对

    # nova keypair-add mykey > oskey.priv
    
在dashboard中launch一个instance，记得勾选密钥对

登录

    # ssh -i oskey.pric root@192.168.100.2
    
如果没有问题说明配置成功了！



[China — cn.pool.ntp.org]: http://www.pool.ntp.org/zone/cn
[sample_data.sh]: https://github.com/openstack/keystone/blob/master/tools/sample_data.sh
