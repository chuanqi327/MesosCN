## 数人科技单机Mesos-Ubuntu脚本安装帮助文档
### 程序解压
    tar czvf single_mesos.tar.gz && cd dataman
### 集群安装
    ./mesos_dataman install
### 集群启动
    ./mesos_dataman start
### 集群关闭
    ./mesos_dataman stop
### 集群重启
    ./mesos_dataman restart
### 集群查看状态
    ./mesos_dataman status
### 添加top任务
    ./mesos_dataman addtop
### 添加nginx任务
    ./mesos_dataman addweb
### 脚本帮助
    ./mesos_dataman -h | --hlep