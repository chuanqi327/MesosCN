# centos6.6 dataman源单机安装mesos marathon
## 1. 下载执行centos-net-install.sh安装各服务
    
    脚本链接：https://github.com/Dataman-Cloud/operation/blob/master/dataman_shell/mesos_system/centos-net-install.sh

## 2. 修改各服务配置文件

### 2.1 zookeeper
    
	mv /usr/local/zookeeper/conf/zoo_sample.cfg /usr/local/zookeeper/conf/zoo.cfg
	mkdir /usr/local/zookeeper/data
	sed -i 's/dataDir=.*/dataDir=\/usr\/local\/zookeeper\/data/g' /usr/local/zookeeper/conf/zoo.cfg
	

### 2.2 Mesos-Master
参考：https://github.com/Dataman-Cloud/operation/edit/master/%E8%BF%90%E7%BB%B4%E6%96%87%E6%A1%A3/%E6%8A%80%E6%9C%AF%E6%96%87%E6%A1%A3/mesos/mesos%E5%8D%95%E6%9C%BApoc/%E5%8D%95%E6%9C%BA%E5%AE%89%E8%A3%85mesos%E7%B3%BB%E7%BB%9F.md

配置Mesos本身信息

    #配置mesos在zk的使用目录
    echo "zk://localhost:2181/mesos" > "/etc/mesos/zk"
配置Mesos-Master相关信息

    #指定master配置目录
    MESOS_MASTER_CONF_DIR="/etc/mesos-master"
    #指定master的主机名
    echo "10.3.10.17" > $MESOS_MASTER_CONF_DIR/hostname
    #指定master的ip
    echo "0.0.0.0" > $MESOS_MASTER_CONF_DIR/ip
    #副本的仲裁数量的大小（集群配置很重要，本次试验只有1台所以写1）
    echo "1" > $MESOS_MASTER_CONF_DIR/quorum
    #注册表中存储持久性信息的地址
    echo "/var/lib/mesos" > $MESOS_MASTER_CONF_DIR/work_dir
	

### 2.3 Mesos-Slave
配置Mesos-Slave相关信息

    #指定slave配置目录
    MESOS_SLAVE_CONF_DIR="/etc/mesos-slave"
    #指定slave的主机名(这里不能用localhost)
    echo "10.3.10.17" > $MESOS_SLAVE_CONF_DIR/hostname
    #指定slave支持的容器类型
    echo "docker,mesos" > $MESOS_SLAVE_CONF_DIR/containerizers
    #指定slave的ip
    echo "0.0.0.0" > $MESOS_SLAVE_CONF_DIR/ip
    #执行器注册超时时间
    echo "5mins" > $MESOS_SLAVE_CONF_DIR/executor_registration_timeout
    #指定mesos资源控制的内容(这里只有打开对CPU和内存的控制)
    echo "cgroups/cpu,cgroups/mem" > $MESOS_SLAVE_CONF_DIR/isolation

### 2.4 MARATHON

配置Marathon相关信息
    
    ＃创建marathon目录
    mkdir /etc/marathon/conf
    #指定marathon配置目录
    MARATHON_CONF_DIR="/etc/marathon/conf"
    #指定marathon在zk目录路径
    echo "zk://127.0.0.1:2181/marathon" > $MARATHON_CONF_DIR/zk
    #事件订阅模式
    echo "http_callback" > $MARATHON_CONF_DIR/event_subscriber
    #指定marathon主机名
    echo "127.0.0.1" > $MARATHON_CONF_DIR/hostname
    #指定mesos在zk目录路径
    echo "zk://127.0.0.1:2181/mesos" > $MARATHON_CONF_DIR/master

### 2.5 BAMBOO

### 2.5.1 注释ha模版的8080部分，否则该8080端口和marathon自带默认端口冲突

    vim /opt/bamboo/config/haproxy_template.cfg
    
    #注释掉一下模版
    frontend websocket-in
        bind *:8080
        {{ $services := .Services }}
        {{ range $index, $app := .Apps }} {{ if $app.Env.BAMBOO_WEBSOCKET_OPEN }} {{ if hasKey $services $app.Id }} {{ $service := getService $services $app.Id }}
        acl {{ $app.EscapedId }}-websocket-aclrule {{ $service.Acl}}:8080
        use_backend {{ $app.EscapedId }}-websocket-cluster if {{ $app.EscapedId }}-websocket-aclrule
        {{ end }} {{ end }} {{ end }}

        stats enable
        # CHANGE: Your stats credentials
        stats auth admin:admin
        stats uri /haproxy_stats

    {{ range $index, $app := .Apps }} {{ if $app.Env.BAMBOO_WEBSOCKET_OPEN }}
    backend {{ $app.EscapedId }}-websocket-cluster{{ if $app.HealthCheckPath }}
        option httpchk GET {{ $app.HealthCheckPath }}
        {{ end }}
        balance leastconn
        option httpclose
        option forwardfor
        {{ range $page, $task := .Tasks }}
        server {{ $app.EscapedId }}-{{ $task.Host }}-{{ index $task.Ports 1 }} {{ $task.Host }}:{{ index     $task.Ports 1 }} {{ end }}
    {{ end }}
    {{ end }}

### 2.5.2 修改bamboo配置

    /bin/cat > /opt/bamboo/config/production.json<<EOF
    {
      "Marathon": {
    "Endpoint": "http://127.0.0.1:8080"
      },

      "Bamboo": {
    "Endpoint": "http://127.0.0.1:8000",
    "Zookeeper": {
      "Host": "127.0.0.1:2181",
      "Path": "/marathon-haproxy/state",
      "ReportingDelay": 5
    }
      },

      "HAProxy": {
    "TemplatePath": "/opt/bamboo/config/haproxy_template.cfg",
    "OutputPath": "/etc/haproxy/haproxy.cfg",
    "ReloadCommand": "PIDS=`pidof haproxy`; haproxy -f /etc/haproxy/haproxy.cfg -p /var/run/haproxy.pid -sf $PIDS && while ps -p $PIDS; do sleep 0.2; done"
      },

      "StatsD": {
    "Enabled": false,
    "Host": "localhost:8125",
    "Prefix": "bamboo-server.development."
      }
    }
    EOF
    
bamboo-json配置文件基础说明:

- http://127.0.0.1:8080 ＃Marathon地址 
- http://127.0.0.1:8000 ＃Bamboo地址
- 127.0.0.1:2181 #zookeeper地址
- /opt/bamboo/config/haproxy_template.cfg ＃bamboo自带haproxy配置文件模版路径
- /etc/haproxy/haproxy.cfg ＃haproxy配置文件路径
- localhost:8125 ＃StatsD监控地址(需要另行安装)



## 3. 服务操作
### 3.1 mesos-master
    # 启动命令
    nohup mesos-master --zk=zk://localhost:2181/mesos  --ip=0.0.0.0 --work_dir=/var/lib/mesos --quorum=1 --log_dir=/var/log/mesos &>> /var/log/mesos/mesos-master.log &
    #查看进程状态
    ps axuf | grep mesos-master | grep -v grep

### 3.2 mesos-slave
    # 启动命令
    nohup  mesos-slave --master=10.3.10.17:5050 --hostname=10.3.10.17 &>> /var/log/mesos/mesos-slave.log &
    #查看进程状态
    ps axuf | grep mesos-slave | grep -v grep
    
### 3.3 marathon
    # 启动命令
    nohup marathon  &>> /var/log/marathon/marathon.log &
    #查看进程状态
    ps axuf | grep marathon | grep -v grep
    
### 3.4 haproxy
    # 启动命令
    service haproxy start
    #进程状态
    ps axuf | grep haproxy | grep -v grep
    
### 3.5 bamboo
    # 启动命令
    nohup /opt/bamboo/bamboo -config /usr/local/bamboo/config/production.json -log /var/log/bamboo-server.log &>>/var/log/bamboo.log &
### 3.5 zookeeper
    # 启动命令
	/usr/local/zookeeper/bin/zkServer.sh start
	# 检查状态
    ps axuf | grep zookeeper | grep -v grep
	echo stat|curl -s telnet://localhost:2181

## 4. 制作镜像
### 4.1 制作CentOS基础镜像

    参考：http://blog.csdn.net/yuzhihui_no1/article/details/41129985

#### 4.1.1 设置docker镜像源
    
    yum install -y yum-priorities && rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm && rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6 
     

#### 4.1.2 安装 docker-io febootstrap，用来制作centos镜像，到时候会生成个centos的镜像。

    yum -y install docker-io；# 如果没有安装docker，则需要先安装docker  
    service docker start ；# 启动docker  
    yum -y install febootstrap；# 制作docker镜像工具
    
#### 4.1.3 作CentOS镜像文件centos6-image目录

    febootstrap -i bash -i wget -i yum -i iputils -i iproute -i man -i vim -i openssh-server -i openssh-clients -i tar -i gzip centos6 centos6-image http://mirrors.aliyun.com/centos/6/os/x86_64/  
    
#### 4.1.4 复制home目录基础文件到root目录 
这时root目录下没有任何文件，也不没有隐藏的点文件，如：.bash_logout  .bash_profile  .bashrc如果这时制作出来的镜像使用ssh登录，会直接进入根目录下，而一般镜像都是进入root目录下的，所以可以在centos6-image目录的root目录把.bash_logout  .bash_profile  .bashrc这三个文件设置一下。
    
    cd centos6-image && cp etc/skel/.bash* root/
#### 4.1.5 生成最基础的base镜像
    cd centos6-image && tar -c .|docker import - centos6-base  
#### 4.1.6 查看镜像，也可以直接进入centos6-base查看
```
#这个是查看所有生成的镜像  
docker images  ; 
#进终端（没有ssh服务），-i 分配终端，-t表示在前台执行，-d表示在后台运行
docker run -i -t centos:centos6 /bin/bash；   
```
    
#### 4.1.7 根据基础镜像制作jdk7的docker镜像
```
mkdir -p /data/jdk7
/bin/cat > /data/jdk7/dockerfile <<EOF
FROM centos6-base
MAINTAINER jyliu jyliu@dataman-inc.com
# install epel
RUN yum install -y yum-priorities && rpm -ivh http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm && rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-6
# install tomcat
RUN yum update -y
RUN yum install -y java-1.7.0-openjdk
EOF
docker build -t centos6-jdk7-base /data/jdk7/
```
#### 4.1.8 根据jdk7的镜像生成tomcat7镜像
参考：https://github.com/docker-library/tomcat/blob/24e869945b9d9d3f02f979524b8caf989d08ed6c/7-jre7/Dockerfile

```
mkdir -p /data/tomcat7
/bin/cat > /data/tomcat7/dockerfile <<EOF
FROM centos6-jdk7
MAINTAINER jyliu jyliu@dataman-inc.com

ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME"
WORKDIR $CATALINA_HOME

# see https://www.apache.org/dist/tomcat/tomcat-8/KEYS
RUN gpg --keyserver pool.sks-keyservers.net --recv-keys \
05AB33110949707C93A279E3D3EFE6B686867BA6 \
07E48665A34DCAFAE522E5E6266191C37C037D42 \
47309207D818FFD8DCD3F83F1931D684307A10A5 \
541FBE7D8F78B25E055DDEE13C370389288584E7 \
61B832AC2F1C5A90F0F9B00A1C506407564C17A3 \
713DA88BE50911535FE716F5208B0AB1D63011C7 \
79F7026C690BAA50B92CD8B66A3AD3F4F22C4FED \
9BA44C2621385CB966EBA586F72C284D731FABEE \
A27677289986DB50844682F8ACB77FC2E86E29AC \
A9C5DF4D22E99998D9875A5110C01C5A2F6059E7 \
DCFD35E0BF8CA7344752DE8B6FB21E8933C60243 \
F3A04C595DB5B6A5F1ECA43E3B7BBB100D811BBE \
F7DA48BB64BCB84ECBA7EE6935CD23C10D498E23

ENV TOMCAT_MAJOR 7
ENV TOMCAT_VERSION 7.0.63
ENV TOMCAT_TGZ_URL https://www.apache.org/dist/tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz

RUN set -x \
	&& curl -fSL "$TOMCAT_TGZ_URL" -o tomcat.tar.gz \
	&& curl -fSL "$TOMCAT_TGZ_URL.asc" -o tomcat.tar.gz.asc \
	&& gpg --verify tomcat.tar.gz.asc \
	&& tar -xvf tomcat.tar.gz --strip-components=1 \
	&& rm bin/*.bat \
	&& rm tomcat.tar.gz*

EXPOSE 8080
CMD ["catalina.sh", "run"]
EOF

docker build -t centos6-jdk7-base /data/tomcat7/

```

## 5. marathon添加APP
### 5.1 编写添加APP脚本 
vi dataman-tomcat-test.sh

```
curl -v -X POST http://127.0.0.1:8080/v2/apps -H Content-Type:application/json -d \
'{
      "id": "dataman-tomcat-test",
      "cmd": "",
      "cpus": 0.1,
      "mem": 128.0,
      "instances": 3,
      "container": {
                     "type": "DOCKER",
                     "docker": {
                                     "image": "centos6-tomcat7-base:laster",
                                     "network": "BRIDGE",
                                     "portMappings": [
                                                       { "containerPort": 8080, "hostPort": 0, "servicePort": 10000, "protocol": "tcp" }
                                                     ]
                                }
                   },
      "healthChecks": [
                        { "protocol": "HTTP",
                          "portIndex": 0,
                          "path": "/",
                          "gracePeriodSeconds": 5,
                          "intervalSeconds": 20,
                          "maxConsecutiveFailures": 3 }
                      ]
}'
```
### 5.2 执行dataman-tomcat-test.sh
	sh dataman-tomcat-test.sh

## 6. 查看marathon mesos状态
    参考：https://github.com/Dataman-Cloud/operation/blob/master/%E8%BF%90%E7%BB%B4%E6%96%87%E6%A1%A3/%E6%8A%80%E6%9C%AF%E6%96%87%E6%A1%A3/mesos/mesos%E5%8D%95%E6%9C%BApoc/%E5%8D%95%E6%9C%BA%E5%AE%89%E8%A3%85mesos%E7%B3%BB%E7%BB%9F.md

## 7. 故障处理：
bamboo 更新haproxy 配置文件后提示下面错误 
    [ALERT] 211/103821 (3365) : parsing [/etc/haproxy/haproxy.cfg:14] : unknown keyword 'ssl-default-bind-options' in 'global' section
原因是centos6默认安装的haproxy版本为1.5.2，  'ssl-default-bind-options' in 'global' 是在1.5.7中加入的参数，需要的话可以升级haproxy版本，在这里是在bamboo模板里注释掉了。
​
