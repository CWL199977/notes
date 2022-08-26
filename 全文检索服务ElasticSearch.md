## 1.ElasticSearch 介绍

>1、elasticsearch是一个基于Lucene的分布式全文检索服务器。
>
> 2、elasticsearch隐藏了Lucene的复杂性，对外提供Restful 接口来操作索引、搜索。

## 2.安装 ElasticSearch
### 2.1.环境需求

1、jdk必须是jdk1.8.0_131以上版本。

2、ElasticSearch 需要至少4096 的线程池、65536个文件创建权限和 262144字节以上空间的虚拟内存才能正常启动，所以需要为虚拟机分配至少1.5G以上的内存

3、从5.0开始，ElasticSearch 安全级别提高了，不允许采用root帐号启动

4、Elasticsearch的插件要求至少centos的内核要3.5以上版本

### 2.2.安装ES

#### 2.2.1.下载

ElasticSearch官网：https://www.elastic.co/cn/

#### 2.3创建用户

从5.0开始，ElasticSearch 安全级别提高了，不允许采用root帐号启动，所以我们要添加一个用户。

##### 1.创建elk 用户组

```sh
groupadd elk
```

##### 2.创建用户admin

```sh
useradd admin
passwd admin
```

##### 3.将admin用户添加到elk组

```sh
usermod -G elk admin
```

##### 4.查看当前用户组

```java
groups
```

##### 5.为用户分配权限

```sh
#chown将指定文件的拥有者改为指定的用户或组 -R处理指定目录以及其子目录下的所有文件
chown -R admin:elk /usr/upload
chown -R admin:elk /usr/local
```

##### 6.切换用户：
```
su admin
```

#### 2.4.安装

ES是Java开发的应用，解压即安装：

```sh
tar -zxvf elasticsearch-6.2.3.tar.gz -C /usr/local
```

#### 2.5.ES目录结构

```tex
bin 目录：可执行文件包
config 目录：配置相关目录
lib 目录：ES 需要依赖的 jar 包，ES 自开发的 jar 包
logs 目录：日志文件相关目录
modules 目录：功能模块的存放目录，如aggs、reindex、geoip、xpack、eval
plugins 目录：插件目录包，三方插件或自主开发插件
data 目录：在 ES 启动后，会自动创建的目录，内部保存 ES 运行过程中需要保存的数据。
```

#### 2.6.配置文件

ES安装目录config中配置文件如下：

-  elasticsearch.yml：用于配置Elasticsearch运行参数 

- jvm.options：用于配置Elasticsearch JVM设置 

- log4j2.properties：用于配置Elasticsearch日志

#### 2.7elasticsearch.yml

本项目配置如下：

```yaml
cluster.name: power_shop
node.name: power_shop_node_1
network.host: 0.0.0.0
http.port: 9200
transport.tcp.port: 9300
discovery.zen.ping.unicast.hosts: ["192.168.94.135:9300", "192.168.94.136:9300"]
path.data: /usr/local/elasticsearch-6.2.3/data
path.logs: /usr/local/elasticsearch-6.2.3/logs
http.cors.enabled: true
http.cors.allow-origin: /.*/
```

**注意意path.data和path.logs路径配置正确。**

常用的配置项如下：

```
cluster.name:
       配置elasticsearch的集群名称，默认是elasticsearch。建议修改成一个有意义的名称。   
node.name:
      节点名，通常一台物理服务器就是一个节点，es会默认随机指定一个名字，建议指定一个有意义的名称，方便管理一个或多个节点组成一个cluster集群，集群是一个逻辑的概念，节点是物理概念，后边章节会详细介绍。
path.data:
       设置索引数据的存储路径，默认是es_home下的data文件夹，可以设置多个存储路径，用逗号隔开。      
path.logs:
       设置日志文件的存储路径，默认是es_home下的logs文件夹         
network.host:  
       设置绑定主机的ip地址，设置为0.0.0.0表示绑定任何ip，允许外网访问，生产环境建议设置为具体的ip。   
http.port: 9200
       设置对外服务的http端口，默认为9200。      
transport.tcp.port: 9300 
       集群结点之间通信端口      
discovery.zen.ping.unicast.hosts:[“host1:port”, “host2:port”, “…”]  
       设置集群中master节点的初始列表。
discovery.zen.ping.timeout: 3s  
       设置ES自动发现节点连接超时的时间，默认为3秒，如果网络延迟高可设置大些。
http.cors.enabled：
	   是否支持跨域，默认为false
http.cors.allow-origin：
	   当设置允许跨域，默认为*,表示支持所有域名
```

#### 2.8 jvm.options

设置最小及最大的JVM堆内存大小：

在jvm.options中设置 -Xms和-Xmx：

- 1 两个值设置为相等

- 2 将`Xmx` 设置为不超过物理内存的一半。

> 默认内存占用太多了，我们调小一些：

```properties
-Xms512m
-Xmx512m
```

## 3.启动ES

### 3.1启动和关闭

#### 1、启动

```sh
cd /usr/local/elasticsearch-6.2.3/bin

./elasticsearch
#或
./elasticsearch -d            
```

#### 2、关闭

```sh
ps-ef|grep elasticsearch

kill -9 pid
```

### 3.2.解决文件创建权限问题

```
[1]: max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]
```

> Linux 默认来说，一般限制应用最多创建的文件是 4096个。但是 ES 至少需要 65536 的文件创建> 权限。我们用的是admin用户，而不是root，所以文件权限不足。

**使用<font color=red>root</font>用户修改配置文件:**

```
vim /etc/security/limits.conf
```

**追加下面的内容：**

```nginx
* soft nofile 65536
* hard nofile 65536
```

### 3.3.解决线程开启限制问题

```
[2]: max number of threads [3795] for user [es] is too low, increase to at least [4096]
```

​	默认的 Linux 限制 root 用户开启的进程可以开启任意数量的线程，其他用户开启的进程可以开启1024 个线程。必须修改限制数为4096+。因为 ES 至少需要 4096 的线程池预备。

​	如果虚拟机的内存是 1G，最多只能开启 3000+个线程数。至少为虚拟机分配 1.5G 以上的内存。

**使用<font color=red>root</font>用户修改配置：**

```sh
vim /etc/security/limits.conf
```

**追加：**

```nginx
* soft nproc  4096
* hard nproc  4096
```

### 3.4.解决虚拟内存问题

```
[3]: max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]
```

**ES 需要开辟一个 262144字节以上空间的虚拟内存。Linux 默认不允许任何用户和应用直接开辟虚拟内存。**

```sh
vim /etc/sysctl.conf
```

**追加下面内容：**

```sh
vm.max_map_count=655360 #限制一个进程可以拥有的VMA(虚拟内存区域)的数量
```

**然后执行命令，让sysctl.conf配置生效：**

```sh
sysctl -p
```
