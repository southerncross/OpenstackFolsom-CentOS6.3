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
PS：上面3条grant指令最好都写上，否则也许会出错，这里假设控制节点的hostname是Ops146

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



