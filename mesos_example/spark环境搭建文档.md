# 数人科技spark环境搭建文档
    文档信息
    创建人 张馨文
    邮件地址 xwzhang@dataman-inc.com
    建立时间 2015年8月06号
    更新时间 2015年8月06号
### 1. 搭建Mesos基础环境
### 2. 部署spark运行环境在Mesos-slave上以及开发机上
    sudo apt-get -y update && sudo apt-get install -y openjdk-7-jre openjdk-7-jdk
    sudo apt-get install -y scala
    sudo apt-get install -y libcurl4-nss-dev
### 3.部署spark tar包下载服务器
    在spark官网下载spark的tar包
    tar包版本指定为spark1.4.1, hadoop2.6
    wget http://mirror.cogentco.com/pub/apache/spark/spark-1.4.1/spark-1.4.1-bin-hadoop2.6.tgz

    安装nginx服务
    sudo apt-get install -y nginx 
    修改配置/etc/nginx/sites-enabled/default
    添加：
    location /spark {
        alias /usr/local/share/;
        autoindex on;
    }
    其中/usr/local/share/为在nginx服务器中存放tar包的路径, /spark为访问站点的相对路径


