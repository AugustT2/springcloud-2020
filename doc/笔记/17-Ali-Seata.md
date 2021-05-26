# 简介
### 解决问题
1. 分布式前Java服务与数据库1->1
2. 分布式后 1->1,1>多,多->多
保证多个服务之间的数据一致性
### 是什么
Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。
### 官网
http://seata.io/zh-cn/
### 处理过程
<img src="imgs/seata.png">

###### 一ID+三组件
1. id
全局唯一的事务ID
2. 3组件
    1. TC - 事务协调者(Transaction Coordinator)
    维护全局和分支事务的状态，驱动全局事务提交或回滚。
    2. TM - 事务管理器
    定义全局事务的范围：开始全局事务、提交或回滚全局事务。
    3. RM - 资源管理器
    管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。
###### 处理过程
1. TM向TC申请开启一个全局事务，全局事务创建成功并生成一个全局唯一的XID
2. XID在微服务调用链路的上下文中传播
3. RM向TC注册分支事务，将其纳入XID对应全局事务的管辖
4. TM 向 TC 发起针对 XID 的全局提交或回滚请求
5. TC 调度 XID 下管辖的全部分支事务完成提交或回滚请求

# 安装
1. 下载
https://github.com/seata/seata/releases
2. 找到conf下的 file.conf ，修改前请备份
将 mode 改为 db,代表将日志存储到数据库
修改数据库账号密码端口为自己的数据库，同时将service标签下的组有default改为自定义组如vgroup_mapping.fsp_tx_group = "default"
找到 register.conf
将 registry 与 config 里的 type均改为nacos
同时修改两者下面的 nacos信息，serverAddr = "localhost:8848"
3. 创建数据库 seata
4. 执行conf文件夹下的db_store.sql文件（第5步是undo日志表的sql，不用管，我们这一步已经有了）
5. 数据库加载文件
    查看RANDME.MD server 对应网址即可
    1. https://github.com/seata/seata/tree/develop/script/client下db中的mysql
6. cmd模式下执行seata-server.sh文件（C:\Users\eding3\Downloads\seata\bin>seata-server.bat，启动前先启动nacos），看到extension by class[io.seata.discovery.registry.nacos.NacosRegistryProvider代表启动ok。

# 实验
### 数据库
1. 创建数据库（sql看第2点）
    1. create database seata_order;订单
    2. create database seata_storage;库存
    3. create database seata_account;账户信息
2. 建库和建表sql
```sql
Drop
Database if exists seata_order;
create
database seata_order;

Drop
Database if exists seata_storage;
create
database seata_storage;

Drop
Database if exists seata_account;
create
database seata_account;

DROP TABLE IF EXISTS seata_order.t_order;
CREATE TABLE seata_order.t_order
(
    `id`         BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `user_id`    BIGINT(11) DEFAULT NULL COMMENT '用户id',
    `product_id` BIGINT(11) DEFAULT NULL COMMENT '产品id',
    `count`      INT(11) DEFAULT NULL COMMENT '数量',
    `money`      DECIMAL(11, 0) DEFAULT NULL COMMENT '金额',
    `status`     INT(1) DEFAULT NULL COMMENT '订单状态：0：创建中; 1：已完结'
) ENGINE = INNODB
  AUTO_INCREMENT = 7
  DEFAULT CHARSET = utf8;

DROP TABLE IF EXISTS seata_storage.t_storage;
CREATE TABLE seata_storage.t_storage
(
    `id`         BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY,
    `product_id` BIGINT(11) DEFAULT NULL COMMENT '产品id',
    `total`      INT(11) DEFAULT NULL COMMENT '总库存',
    `used`       INT(11) DEFAULT NULL COMMENT '已用库存',
    `residue`    INT(11) DEFAULT NULL COMMENT '剩余库存'
) ENGINE = INNODB
  AUTO_INCREMENT = 2
  DEFAULT CHARSET = utf8;

DROP TABLE IF EXISTS seata_account.t_account;
CREATE TABLE seata_account.t_account
(
    `id`      BIGINT(11) NOT NULL AUTO_INCREMENT PRIMARY KEY COMMENT 'id',
    `user_id` BIGINT(11) DEFAULT NULL COMMENT '用户id',
    `total`   DECIMAL(10, 0) DEFAULT NULL COMMENT '总额度',
    `used`    DECIMAL(10, 0) DEFAULT NULL COMMENT '已用余额',
    `residue` DECIMAL(10, 0) DEFAULT '0' COMMENT '剩余可用额度'
) ENGINE = INNODB
  AUTO_INCREMENT = 2
  DEFAULT CHARSET = utf8;


INSERT INTO seata_storage.t_storage(`id`, `product_id`, `total`, `used`, `residue`)
VALUES ('1', '1', '100', '0', '100');

INSERT INTO seata_account.t_account(`id`, `user_id`, `total`, `used`, `residue`)
VALUES ('1', '1', '1000', '0', '1000');



-- the table to store seata xid data
-- 0.7.0+ add context
-- you must to init this sql for you business databese. the seata server not need it.
-- 此脚本必须初始化在你当前的业务数据库中，用于AT 模式XID记录。与server端无关（注：业务数据库）
-- 注意此处0.3.0+ 增加唯一索引 ux_undo_log
DROP TABLE IF EXISTS seata_order.undo_log;
CREATE TABLE seata_order.undo_log
(
    `id`            bigint(20) NOT NULL AUTO_INCREMENT,
    `branch_id`     bigint(20) NOT NULL,
    `xid`           varchar(100) NOT NULL,
    `context`       varchar(128) NOT NULL,
    `rollback_info` longblob     NOT NULL,
    `log_status`    int(11) NOT NULL,
    `log_created`   datetime     NOT NULL,
    `log_modified`  datetime     NOT NULL,
    `ext`           varchar(100) DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8;


DROP TABLE IF EXISTS seata_storage.undo_log;
CREATE TABLE seata_storage.undo_log
(
    `id`            bigint(20) NOT NULL AUTO_INCREMENT,
    `branch_id`     bigint(20) NOT NULL,
    `xid`           varchar(100) NOT NULL,
    `context`       varchar(128) NOT NULL,
    `rollback_info` longblob     NOT NULL,
    `log_status`    int(11) NOT NULL,
    `log_created`   datetime     NOT NULL,
    `log_modified`  datetime     NOT NULL,
    `ext`           varchar(100) DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8;


DROP TABLE IF EXISTS seata_account.undo_log;
CREATE TABLE seata_account.undo_log
(
    `id`            bigint(20) NOT NULL AUTO_INCREMENT,
    `branch_id`     bigint(20) NOT NULL,
    `xid`           varchar(100) NOT NULL,
    `context`       varchar(128) NOT NULL,
    `rollback_info` longblob     NOT NULL,
    `log_status`    int(11) NOT NULL,
    `log_created`   datetime     NOT NULL,
    `log_modified`  datetime     NOT NULL,
    `ext`           varchar(100) DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE = InnoDB
  AUTO_INCREMENT = 1
  DEFAULT CHARSET = utf8;
```
### 建模块
###### seata-order-service2001
1. pom
```xml
<!-- seata -->
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-seata</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.seata</groupId>
            <artifactId>seata-all</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<!-- 引入与自己版本相同的 -->
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-all</artifactId>
    <version>1.2.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

2. yml 
3. file.conf和registry.conf拷贝到项目的资源目录
4. 测试http://localhost:2001/order/create?userId=1&productId=1&count=10&money=100
5. accout加超时，测试出现事务问题，加上@GlobalTransactional(name = "fsp-create-order",rollbackFor = Exception.class)解决事务问题。

原理：

1. TC、TM、RM理解

![image-20210526151856309](imgs\image-20210526151856309.png)

2.流程整理(二段提交)

![image-20210526151950700](\imgs\seata服务流程)

3.AT模式

- 一阶段加载

![image-20210526153506835](imgs\一阶段加载)



- 二阶段提交

![](imgs\二阶段提交)

- 二阶段回滚

![image-20210526154039142](C:\java\springcloud-2020\doc\笔记\imgs\二阶段回滚)









########################## 不用看了

1. 依赖
```xml
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>io.seata</groupId>
                    <artifactId>seata-spring-boot-starter</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <!-- 引入与自己版本相同的 -->
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>1.2.0</version>
        </dependency>
```
2. yml

3. config.txt nacos-config.sh 上传配置
