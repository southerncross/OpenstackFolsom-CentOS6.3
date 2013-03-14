# OpenStack folsom 在CentOS6.3上的配置

## 安装NTP服务

### 安装NTP

    # yum install -y ntp 

### 修改/etc/ntp.conf文件，限定为中国地区的ntp服务器（[China — cn.pool.ntp.org]）

### 启动ntp服务

    # service ntpd start  
    # chkconfig ntpd on

PS: 理想做法是控制节点与某个ntp服务器同步时间，计算节点向控制节点同步时间
