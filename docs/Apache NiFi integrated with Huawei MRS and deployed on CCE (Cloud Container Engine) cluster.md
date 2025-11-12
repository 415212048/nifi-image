# 1.版本说明
MRS	3.5.0-LTS

NIFI	nifi:1.28.1
# 2.场景介绍
基于Apache NIFI开源容器镜像，与华为MapReduce服务(MRS)深度集成，对接MRS Kafka、HDFS、Hive三大核心组件：通过Kafka组件实现高吞吐量的实时数据采集与消息队列管理；利用HDFS组件构建可靠的海量数据存储层；基于Hive组件完成结构化数据的分析与查询。
# 3.前置条件
## 3.1	源码编译
### 3.1.1	nifi. properties配置文件和容器环境变量映射关系
官方提供的docker镜像，容器化部署时，nifi.properties文件内容是通过源码中的start.sh脚本，将容器中配置的环境变量，映射到配置文件中，定制化需要查看脚本中是否有对应的映射关系，如果没有需要修改后手动编译
示例：nifi. properties添加kerberos认证
修改源码的nifi-docker/dockerhub/sh/start.sh文件
 
 
源码可进容器中/opt/nifi/scripts/start.sh
### 3.1.2	源码编译制作docker镜像
在nifi的根目录下执行mvn clean install -T 2C -DskipTests进行源码编译

-T 2C：(可选) 启用多线程编译（每 CPU 核心 2 个线程），加快速度。
环境变量添加对应的值进行映射
### 3.1.3	Docker镜像制作
源码编译完成后，在/nifi/nifi-assembly/target目录下可以看到编译后的zip安装包，复制到/nifi/nifi-docker/dockerhub目录下，执行以下命令制作成docker镜像：
docker build --build-arg NIFI_VERSION=1.28.1 --build-arg UID=1001 --build-arg GID=1001 --tag apache/nifi:1.28.1-v1 .

 

 
### 3.1.4	添加适配MRS相关插件包和文件
添加HDFS相关的文件和扩展包

①	服务器上新建目录：
mkdir /root/nifi-modify

②	拷贝MRS集群客户端HDFS配置文件hdfs-site.xml及core-site.xml至新建的目录下：
MRS HDFS客户端配置文件hdfs-site.xml和core-site.xml目录：
/hadoopclient/HDFS/hadoop/etc/hadoop

③	修改nifi/conf/hdfs-site.xml文件，删除如下内容：
<property>
<name>dfs.client.failover.proxy.provider.hacluster</name>
<value>org.apache.hadoop.hdfs.server.namenode.ha.AdaptiveFailoverProxyProvider</value>
</property>

④	修改/nifi/conf/core-site.xml文件，fs.defaultFS配置项hacluster修改为HDFS namenode(主)IP+端口（dfs.namenode.rpc.port）：
<property>
<name>fs.defaultFS</name>
<value>hdfs://192.168.0.***:8020</value>
</property>
其中，Namenode主节点IP地址及端口号，可以在MRS Manager查询，示例如下：
 
 
hadoop-plugins-*.jar包添加
在之前源码编译后的zip包中，在lib目录中找到nifi-hadoop-nar-1.28.1.nar包，复制到/root/nifi-modify目录下，将hadoop-plugins-*.jar添加到nifi-hadoop-nar-1.28.1.nar包下的nifi-hadoop-nar-1.28.1.nar-unpacked/NAR-INF/bundled-dependencies目录中

添加MRS Hive相关的文件和扩展包
获取客户的nifi-hive3-nar-1.28.1.nar和nifi-hive-services-api-nar-1.28.1.nar，在编译后的zip安装包的lib目录下找到nifi-hive3-nar-1.28.1.nar，将3个nar包复制到/root/nifi-modify目录下
获取MRS Hive Beeline客户端zookeeper-*.jar及zookeeper-jute-*.jar，将这两个jar包替换到nifi-hive3-nar-1.28.1.nar包中nifi-hive3-nar-1.28.1.nar-unpacked/NAR-INF/bundled-dependencies目录下

添加MRS Kfka相关的文件和扩展包
在编译后的zip安装包的lib目录下找到nifi-kafka-2-0-nar-1.28.1.nar包，复制到/root/nifi-modify目录下
获取MRS Kafka客户端Kafka-clients-*.jar及zookeeper-*.jar，将这两个jar包替换到nifi-kafka-2-0-nar-1.28.1.nar包的nifi-kafka-2-0-nar-1.28.1.nar-unpacked/NAR-INF/bundled-dependencies目录下

修改Nifi /conf/bootstrap.conf文件
在编译后的zip安装包的conf目录下获取bootstrap.conf，复制到/root/nifi-modify目录下，修改添加jvm系统属性参数，指定jaas配置信息等：
```txt
java.arg.18=-Djava.security.auth.login.config=/opt/nifi-1.28.1/conf/jaas.conf
java.arg.19=-Dsun.security.krb5.debug=false
java.arg.20=-Dkerberos.domain.name= hadoop.8DAD1BC1_426B_45B3_810B_******.COM
java.arg.21=-Djava.security.krb5.conf=/opt/wuan/krb5.conf
```

添加完成后，应包含如下文件
 

编写Dockerfile，制作定制化镜像
```bash
# 基础镜像：源码修改后的Apache NiFi 1.28.1
FROM apache/nifi:1.28.1-v1

# 定义 NiFi 安装根目录
ENV NIFI_BASE_DIR=/opt/nifi
ENV NIFI_HOME=${NIFI_BASE_DIR}/nifi-current
# 切换为 nifi 用户（与基础镜像保持一致）
USER nifi

# 工作目录
WORKDIR ${NIFI_HOME}

# KAFKA 连接配置
COPY ./bootstrap.conf /opt/nifi/nifi-current/conf/bootstrap.conf
RUN rm -f /opt/nifi/nifi-current/lib/nifi-kafka-2-6-nar-1.28.1.nar
RUN rm -f /opt/nifi/nifi-current/lib/nifi-kafka-2-0-nar-1.28.1.nar
RUN rm -f /opt/nifi/nifi-current/lib/nifi-kafka-1-0-nar-1.28.1.nar
COPY ./nifi-kafka-2-6-nar-1.28.0.nar /opt/nifi/nifi-current/lib
COPY ./nifi-kafka-2-0-nar-1.28.0.nar /opt/nifi/nifi-current/lib
COPY ./nifi-kafka-1-0-nar-1.28.0.nar /opt/nifi/nifi-current/lib

# HDFS 连接配置
COPY ./core-site.xml /opt/nifi/nifi-current/conf/core-site.xml
COPY ./hdfs-site.xml /opt/nifi/nifi-current/conf/hdfs-site.xml
RUN rm -f /opt/nifi/nifi-current/lib/nifi-hadoop-nar-1.28.1.nar
COPY ./nifi-hadoop-nar-1.28.1.nar /opt/nifi/nifi-current/lib

# HIVE 连接配置
 COPY ./nifi-hive-services-api-nar-1.28.1.nar /opt/nifi/nifi-current/lib
 COPY ./nifi-hive3-nar-1.28.1.nar /opt/nifi/nifi-current/lib

# 保留原启动命令（无需自定义脚本，权限已通过 umask 确保）
CMD ["bin/nifi.sh", "run"]
```
制作镜像上传到SWR
```bash
docker build -t apache/nifi:1.28.1-v2 .

sudo docker tag {镜像名称}:{版本名称} swr.ap-southeast-4.myhuaweicloud.com/{组织名称}/{镜像名称}:{版本名称}

sudo docker push swr.ap-southeast-4.myhuaweicloud.com/{组织名称}/{镜像名称}:{版本名称}
```
# 4	在CCE上集群部署
## 4.1	NiFi配置Kerberos认证
①	在FusionInsight Manager下载认证用户（例如“wuan”）的配置文件user.keytab，krb5.conf，并一起存入NIFI所在机器的某个目录；
 
 

## 4.2	创建配置项和秘钥
1. 将4.1章节中获取的user.keytab，krb5.conf配置成Secret
2. 准备jaas.conf,创建成configmap
```txt
Client {
com.sun.security.auth.module.Krb5LoginModule required
useKeyTab=true
principal="wuan@8DAD1BC1_426B_45B3_810B_**********.COM"
keyTab="/opt/wuan/user.keytab"
useTicketCache=false
storeKey=true
debug=true;
};
KafkaClient {
com.sun.security.auth.module.Krb5LoginModule required
useKeyTab=true
principal="wuan@8DAD1BC1_426B_45B3_810B_**********.COM "
keyTab="/opt/wuan/user.keytab"
useTicketCache=false
serviceName="kafka"
storeKey=true
debug=true;
};
```
4. 创建nifi-config，添加以下数据
键：nifi-ext.properties
值：
```txt
nifi.flow.configuration.file=./data/flow.xml.gz
nifi.flow.configuration.json.file=./data/flow.json.gz
nifi.flow.configuration.archive.dir=./data/archive
nifi.database.directory=./data/database_repository
nifi.flowfile.repository.directory=./data/flowfile_repository
nifi.content.repository.directory.default=./data/content_repository
nifi.provenance.repository.directory.default=./data/provenance_repository
nifi.templates.directory=./data/templates
nifi.authorizer.configuration.file=./data/authorizer.xml
nifi.kerberos.krb5.file=/etc/kerberos/krb5.conf
nifi.kerberos.service.principal=wuan@A537368E_6E70_4279_BB36_**********.COM
nifi.kerberos.keytab.location=/etc/kerberos/user.keytab
```
## 4.3	集群部署
### 4.3.1	创建zookeeper集群
选择有状态负载来部署zookeeper
 
 
 
脚本
```bash
# 1. 验证PVC是否挂载成功（/data目录必须存在，否则退出）if [ ! -d "/data" ]; then  echo "ERROR: /data directory not found! PVC mount failed."  exit 1fiecho "PVC mounted successfully: /data (permissions: $(ls -ld /data))"# 2. 提取节点ID（从Pod名称zookeeper-cluster-0中提取第3段→0）# 若Pod名称格式变更（如zk-0），需修改awk参数为'{print $2}'ID=$(echo $HOSTNAME | awk -F '-' '{print $3}')echo "Extracted node ID: $ID"# 3. 生成myid文件（强制写入/data目录，与zoo.cfg的dataDir一致）echo $ID > /data/myid# 验证myid生成结果if [ -f "/data/myid" ]; then  echo "Generated myid: $(cat /data/myid) (path: /data/myid)"else  echo "ERROR: Failed to create myid file!"  exit 1fi# 4. 验证zoo.cfg配置文件是否挂载成功if [ ! -f "/conf/zoo.cfg" ]; then  echo "ERROR: /conf/zoo.cfg not found! ConfigMap mount failed."  exit 1fiecho "zoo.cfg mounted successfully: $(cat /conf/zoo.cfg | grep -E 'dataDir|server')"# 5. 启动ZooKeeper（前台启动，确保容器不退出）zkServer.sh start-foreground
```
 
 
创建实例间发现服务和集群内访问服务
 

### 4.3.2	创建nifi集群
选择有状态负载来部署nifi
1.	基本信息
 
2.	环境变量
 
 
3.	数据存储
 
 
 
4.	创建存储卷声明(PVC)
 
5.	安全设置
 
6.	实例间发现服务配置
名称：headless-nifi-v1
访问端口：8080 -> 8080		8082->8082
7.	服务配置
服务名称：nifi-v1-97973
访问类型：负载均衡
访问端口：8080 -> 8080
 
# 5	集成MRS验证

## 5.1	NiFi配置Kerberos认证
登录NiFi网页界面，进入对应的Process Group中，右键点击Configure，进入Configuration，选择“CONTROLLER SERVICES”页签，点击**+**按钮添加Controller Service，Filter输入keytab过滤并选择KeytabCredentialsService，点击**ADD**添加：
 
点击**齿轮**图标进行KeytabCredentialsService配置，其中Keytab及Principal与nifi.properties中配置的Kerberos及Service principal保持一致：
 
点击**闪电**图标并ENABLE生效，保存KeytabCredentialsService：
 
当其他Processor，例如PublishKafka_2_0 1.28.1配置Kerberos Credentials Service指定KeytabCredentialsService即可关联对应的user.keytab配置项。

## 5.2	NiFi连接MRS HDFS
--处理器GetFile配置 
--处理器PutHDFS配置：
 
验证结果：
 

## 5.3	NiFi连接MRS Hive
①	Hive3ConnectionPool配置步骤：
登录NiFi网页界面，或进入对应的Process Group中，右键点击Configure，进入Configuration，选择“CONTROLLER SERVICES”页签，点击**+**按钮添加Controller Service，Filter输入hive过滤并选择Hive3ConnectionPool，点击**ADD**添加：
 
点击**齿轮**图标进行配置：
 
 
SelectHive3QL配置如下：
 

验证结果：在Hive看到HQL请求成功
 
## 5.4	NiFi连接MRS Kfka
①	驱动器GetFile配置如下：
 
②	驱动器PublishKafka_2_0配置如下：
 
 
验证结果：
 
# 6	集群监控运维
## 6.1	插件安装
创建CCE集群时需要安装kube-prometheus-stack、CCE Log Collector、CCE Node Problem Detector插件 
配置CCE采集的Prometheus数据上报AOM实例
 
## 6.2	资源监控
进入CCE控制台页面，菜单栏中选择“ 云原生观测 > 监控中心”， 即可从集群、节点、工作负载、Pod、事件、仪表盘五个维度去查看集群的状况
### 6.2.1	集群概览
 
 

 
### 6.2.2	节点概览
 
 
 
### 6.2.3	工作负载概览
 
### 6.2.4	Pod概览
 

### 6.2.5	事件概览
 
### 6.2.6	仪表盘视图
 

## 6.3	告警设置
进入CCE控制台页面，菜单栏中选择“ 云原生观测 > 告警中心”， 选择告警规则，开启告警中心，配置告警规则，默认使用AOM中的CCE告警模板，联系组手动创建，目前支持短信，邮件，钉钉，飞书和企业微信进行消息通知。
 
 
 除自带的CCE告警规则外，需要根据业务进行定制化的规则，可添加自定义告警
 
## 6.4	日志转储
### 6.4.1	日志接入
1.	登录云日志服务控制台。
2.	在左侧导航栏中，选择“日志接入 > 接入中心”，在类型下方选择“运行环境”或“云服务”，鼠标悬浮在CCE卡片上，单击“接入日志（LTS）”进入配置页面。
图1 进入CCE接入配置
 
或在左侧导航栏中，选择“日志接入 > 接入管理”，单击“创建”，在弹出的页面中，在类型下方选择“运行环境”或“云服务”，鼠标悬浮在CCE卡片上，单击“接入日志（LTS）”进入配置页面。详细操作可参考云容器引擎CCE应用日志接入LTS
### 6.4.2	日志转储
在云日志服务 LTS左侧导航栏中，选择“日志转储”，点击右上角“配置转储”，进行配置，选择6.3.1章节中配置好的日志组和日志流，存入到对应的OBS中。
 


# 7	问题点
## 7.1	环境变量和nifi.properties映射关系
•	容器化NiFi采用环境变量映射机制，通过CCE平台注入环境变量来动态修改nifi.properties配置
•	核心映射逻辑实现在镜像的start.sh启动脚本中

## 7.2	nifi配置文件不能通过confMap挂载替换
现象：通过ConfigMap挂载nifi.properties到conf目录时，容器启动失败，提示文件只读错误
根本原因：
•	NiFi启动时，start.sh脚本会动态写入配置数据到nifi.properties
•	ConfigMap挂载的文件在容器内默认为只读模式
•	写入操作与只读挂载产生冲突，导致启动失败
解决方案：
1.	在源码的start.sh中预定义环境变量映射关系
2.	通过环境变量方式传递配置参数
3.	重新编译构建Docker镜像
4.	上传至SWR并使用新镜像部署
## 7.3	nifi连接MRS插件包替换
ECS部署：可直接在work/nar/extensions目录添加MRS扩展包，重启服务后持久化保留
容器部署问题：
•	容器重启后work目录重新生成
•	手动添加的插件包丢失
•	无法实现插件持久化
解决方案：
1.	准备阶段：收集所需jar包和nar包
2.	镜像构建：
- 以源码编译的基础镜像为底座
- 编写Dockerfile将NAR包复制到lib目录
- 构建包含插件的新镜像
3.	部署运行：
- 容器启动时自动加载lib目录下所有NAR包
- 自动生成work/nar/extensions扩展包
- 实现MRS连接插件的持久化使用
核心优势：通过构建时集成替代运行时手动添加，确保插件在容器重启后不丢失。
## 7.4	Nifi在CCE上部署问题
### 7.4.1	Headless Service问题
现象：工作负载自动创建Headless Service后，删除重建会导致Pod间通信失败
根本原因：
- 工作负载（如StatefulSet）在创建时会绑定到特定的Headless Service
- 即使删除后重新创建同名Service，工作负载仍引用原有的Service对象
- Pod网络配置未更新到新的Service实例
解决方案：
1.	在工作负载YAML中更新serviceName字段
2.	指向新建的Headless Service名称
3.	重新部署工作负载使配置生效
补充建议：
- 建议在初始创建时就明确指定Headless Service，避免自动创建
- 升级时注意检查工作负载与Service的绑定关系
- 可通过kubectl get statefulset <name> -o yaml验证当前绑定的Service
### 7.4.2	配置项的问题
敏感属性密钥配置
版本要求：NiFi 1.14.0+必须配置nifi.sensitive.props.key
问题现象：未配置时启动失败，无法解密敏感属性
解决方案：
- 通过环境变量或ConfigMap设置该密钥值
- 建议使用Kubernetes Secret存储敏感密钥
- 确保密钥在集群节点间保持一致

网络地址解析问题
根本原因：HOSTNAME环境变量默认为Pod名称而非IP
影响场景：
- 集群节点间通信配置
- 端口绑定配置
- DNS解析失败导致启动异常
解决建议：
- 使用Pod IP替代Pod名称进行网络配置
- 通过status.podIP环境变量获取实际IP
- 配置文件中避免直接引用HOSTNAME作为网络地址

### 7.4.3	MRS Hive连接问题
Database Connection URL：为jdbc:hive2//连接串，可通过MRS客户端beeline登录获取，示例如下：
 
这个地址是从MRS上的zookeeper中获取hive的kerberos认证信息，如果nifi集群配置的是外部zookeeper，会导致如下错误
 
正确的url地址应该是：jdbc:hive2://<HiveServer2-IP>:<端口>/<数据库名>;auth=KERBEROS;principal=hive/<HiveServer2-IP>@<MRS-REALM>
jdbc:hive2://***:10000/;sasl.qop=auth-conf;auth=KERBEROS;principal=hive/hadoop.a537368e_****_****_****_9ddde13659c8.com@A537368E_****_4279_BB36_****************.COM
### 7.4.4	SASL身份认证问题
当NiFi配置nifi.zookeeper.auth.type=sasl时，Zookeeper服务器必须同步启用SASL认证
双方认证机制需保持一致，否则将导致连接失败
# 8	总结
Nifi集成MRS在CCE上进行部署，主要修改点分为以下三类

1.	源码修改与编译构建
- 预先明确MRS连接所需的全部配置项
- 检查start.sh脚本，确保所有配置项都有正确的环境变量映射
- 对缺失的映射关系手动添加配置
- 完成源码修改后进行完整编译构建
-	生成基础镜像
2.	定制化镜像制作
  - 准备MRS集成所需的插件包和XML配置文件
  - 编写Dockerfile时：
      - 以自编译的基础镜像作为起点
      - 添加COPY指令替换关键配置文件
      - 确保权限和路径设置正确
  - 执行镜像构建，生成MRS定制化镜像
  -	进行基础功能验证
3.	CCE部署配置
   - 网络规划：
      - 配置适当的网络策略
      - 确保Pod间通信畅通
   - 服务发现：
      - 设置正确的服务注册与发现机制
      - 配置必要的DNS解析
   - 部署验证：
      - 检查各组件连通性
      - 验证MRS服务访问正常

