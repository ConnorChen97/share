# 调度工具

### 1. 定时器
- jdk提供的定时器
- Spring中的定时器

### 2. 集群任务框架
- Quartz

### 3. 分布式任务平台
- xxl-job
- Elastic-job(apache shardingsphere-elasticjob)
  - Elastic-lite
  - Elastic-cloud
- Powerjob
- Apache DolphinScheduler

### 4. 工作流调度工具
- Azkaban
- Oozie

### 5. 资源调度工具
- Apache Mesos
- Apache Yarn

## 一.jdk中的定时器
### 1.java.util.Timer 
可以通过创建 java.util.TimerTask 调度任务，在同一个线程中串行执行，相互影响。也就是说，对于同一个 Timer 里的多个 TimerTask 任务，如果一个 TimerTask 任务在执行中，其它 TimerTask 即使到达执行的时间，也只能排队等待。因为 Timer 是串行的，同时存在坑 ，所以后来 JDK 又推出了 ScheduledExecutorService ，Timer 也基本不再使用。

### 2.java.util.concurrent.ScheduledExecutorService 
在 JDK 1.5 新增，基于线程池设计的定时任务类，每个调度任务都会被分配到线程池中并发执行，互不影响。这样，ScheduledExecutorService 就解决了 Timer 串行的问题。

### 缺点
- 它们仅支持按照指定频率，不直接支持指定时间的定时调度，需要我们结合 Calendar 自行计算，才能实现复杂时间的调度。例如说，每天、每周几、2020-12-17 等等。
- 它们是进程级别，而我们为了实现定时任务的高可用，需要部署多个进程。此时需要等多考虑，多个进程下，同一个任务在相同时刻，不能重复执行。
- 项目可能存在定时任务较多，需要统一的管理，此时不得不进行二次封装。

## 二.Spring中的定时器
### 1.Spring Framework 中的 Spring Task 
提供了轻量级的定时任务的实现。   
Spring Task 是 Spring Framework 的模块，引入 spring-boot-starter-web 即可。  
[SpringTask文档](https://docs.spring.io/spring-framework/docs/5.2.x/spring-framework-reference/integration.html#scheduling)  
[异步支持](https://docs.spring.io/spring-framework/docs/5.2.x/spring-framework-reference/integration.html#scheduling-annotation-support)

@Configuration
@EnableScheduling

@Scheduled

Scheduled常用参数：
- cron 属性：Spring Cron 表达式。
- fixedDelay 属性：固定执行间隔，单位：毫秒。注意，以调用完成时刻为开始计时时间。
- fixedRate 属性：固定执行间隔，单位：毫秒。注意，以调用开始时刻为开始计时时间。
不常用参数：
- initialDelay 属性：初始化的定时任务执行延迟，单位：毫秒。
- zone 属性：解析 Spring Cron 表达式的所属的时区。默认情况下，使用服务器的本地时区。
- initialDelayString 属性：initialDelay 的字符串形式。
- fixedDelayString 属性：fixedDelay 的字符串形式。
- fixedRateString 属性：fixedRate 的字符串形式。

application.yml常用配置
```yml
spring:
  task:
    # Spring Task 调度任务的配置，对应 TaskSchedulingProperties 配置类
    scheduling:
      thread-name-prefix: scheduling- # 线程池的线程名的前缀。默认为 scheduling- ，建议根据自己应用来设置
      pool:
        size: 10 # 线程池大小。默认为 1 ，根据自己应用来设置
      shutdown:
        await-termination: true # 应用关闭时，是否等待定时任务执行完成。默认为 false ，建议设置为 true
        await-termination-period: 60 # 等待任务完成的最大时长，单位为秒。默认为 0 ，根据自己应用来设置
```

[spring.task.scheduling对应Spring中的TaskSchedulingProperties](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskSchedulingProperties.java)
[SpringBoot的TaskSchedulingAutoConfiguration自动配置Spring Task](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/task/TaskSchedulingAutoConfiguration.java)

### 2. Spring Boot 2.0 版本，整合了 Quartz 作业调度框架
提供了功能强大的定时任务的实现。

> Spring Framework 已经内置了 Quartz 的整合。Spring Boot 1.X 版本未提供 Quartz 的自动化配置，而 2.X 版本提供了支持。

spring-boot-starter-quartz依赖对Quartz自动化配置

> 只提供了任务调度的功能，不提供管理任务的管理与监控控制台，需要二次封装
> 无法满足管理与监控的需求

## Quartz
Quartz 自带了集群方案。它通过将作业信息存储到关系数据库中，并使用关系数据库的行锁来实现执行作业的竞争，从而保证多个进程下，同一个任务在相同时刻，不能重复执行。

特性：数据库支持，集群，插件，EJB 作业预构建，JavaMail 及其它，支持 cron-like 表达式等等。

组件：
- Scheduler ：调度器
- Trigger ：触发器
- Job ：任务  

[Quartz 入门详解](https://www.jianshu.com/p/7663f0ed486a)   

内存存储
- 配置容易，运行速度快
- 因为调度程序信息是存储在被分配给 JVM 的内存里面，所以，当应用程序停止运行时，所有调度信息将被丢失。另外因为存储到JVM内存里面，所以可以存储多少个 Job 和 Trigger 将会受到限制

数据库存储
- 支持集群，因为所有的任务信息都会保存到数据库中，可以控制事务，还有就是如果应用服务器关闭或者重启，任务信息都不会丢失，并且可以恢复因服务器关闭或者重启而导致执行失败的任务
- 运行速度的快慢取决与连接数据库的快慢

缺点：
- 集群部署依赖MySQL
- 依赖MySQL行锁

## 三.开源调度任务中间件
1. Elastic-Job
2. Apache DolphinScheduler
3. XXL-JOB

> 唯品会的 Saturn 基于 Elastic-Job

### 1. xxl-job
轻量级分布式任务调度平台  
[xxl-job官方文档](https://www.xuxueli.com/xxl-job/)

### 特性
#### 1、 功能强大
1. 简单：支持通过 Web 页面对任务进行 CRUD 操作，操作简单，一分钟上手；
2. 动态：支持动态修改任务状态、启动/停止任务，以及终止运行中任务，即时生效；
3. 事件触发：除了”Cron 方式”和”任务依赖方式”触发任务执行之外，支持基于事件的触发任务方式。调度中心提供触发任务单次执行的 API 服务，可根据业务事件灵活触发。
4. GLUE：提供 Web IDE，支持在线开发任务逻辑代码，动态发布，实时编译生效，省略部署上线的过程。支持 30 个版本的历史版本回溯。 19、脚本任务：支持以 GLUE 模式开发和运行脚本任务，包括 Shell、Python、NodeJS、PHP、PowerShell 等类型脚本;
5. 命令行任务：原生提供通用命令行任务 Handler（Bean任务，”CommandJobHandler”）；业务方只需要提供命令行即可；
6. 任务依赖：支持配置子任务依赖，当父任务执行结束且执行成功后将会主动触发一次子任务的执行, 多个子任务用逗号分隔；
7. 自定义任务参数：支持在线配置调度任务入参，即时生效；
8. 数据加密：调度中心和执行器之间的通讯进行数据加密，提升调度信息安全性；
9. 跨平台：原生提供通用 HTTP 任务Handler（Bean 任务，”HttpJobHandler”）；业务方只需要提供 HTTP 链接即可，不限制语言、平台；
10. 国际化：调度中心支持国际化设置，提供中文、英文两种可选语言，默认为中文；

#### 2、 高性能
1. 分片广播任务：执行器集群部署时，任务路由策略选择”分片广播”情况下，一次任务调度将会广播触发集群中所有执行器执行一次任务，可根据分片参数开发分片任务；
2. 动态分片：分片广播任务以执行器为维度进行分片，支持动态扩容执行器集群从而动态增加分片数量，协同进行业务处理；在进行大数据量业务操作时可显著提升任务处理能力和速度。
3. 调度线程池：调度系统多线程触发调度运行，确保调度精确执行，不被堵塞；
4. 全异步：任务调度流程全异步化设计实现，如异步调度、异步运行、异步回调等，有效对密集调度进行流量削峰，理论上支持任意时长任务的运行；
5. 线程池隔离：调度线程池进行隔离拆分，慢任务自动降级进入 ”Slow” 线程池，避免耗尽调度线程，提高系统稳定性；

#### 3、 高可用
1. 调度中心 HA（中心式）：调度采用中心式设计，“调度中心”自研调度组件并支持集群部署，可保证调度中心 HA；
2. 执行器 HA（分布式）：任务分布式执行，任务”执行器”支持集群部署，可保证任务执行 HA；
3. 路由策略：执行器集群部署时提供丰富的路由策略，包括：第一个、最后一个、轮询、随机、一致性 HASH、最不经常使用、最近最久未使用、故障转移、忙碌转移等；
4. 故障转移：任务路由策略选择”故障转移”情况下，如果执行器集群中某一台机器故障，将会自动 Failover 切换到一台正常的执行器发送调度请求。
5. 阻塞处理策略：调度过于密集执行器来不及处理时的处理策略，策略包括：单机串行（默认）、丢弃后续调度、覆盖之前调度；
6. 任务超时控制：支持自定义任务超时时间，任务运行超时将会主动中断任务；
7. 任务失败重试：支持自定义任务失败重试次数，当任务失败时将会按照预设的失败重试次数主动进行重试；其中分片任务支持分片粒度的失败重试；
8. 一致性：“调度中心”通过DB锁保证集群分布式调度的一致性, 一次任务调度只会触发一次执行；

#### 4、 监控治理
1.注册中心: 执行器会周期性自动注册任务, 调度中心将会自动发现注册的任务并触发执行。同时，也支持手动录入执行器地址；
2.弹性扩容缩容：一旦有新执行器机器上线或者下线，下次调度时将会重新分配任务；
3.任务失败告警；默认提供邮件方式失败告警，同时预留扩展接口，可方便的扩展短信、钉钉等告警方式；
4. 任务进度监控：支持实时监控任务进度；
5. Rolling 实时日志：支持在线查看调度结果，并且支持以 Rolling 方式实时查看执行器输出的完整的执行日志；
6. 邮件报警：任务失败时支持邮件报警，支持配置多邮件地址群发报警邮件；
7. 推送 Maven 中央仓库: 将会把最新稳定版推送到 Maven 中央仓库, 方便用户接入和使用;
8. 运行报表：支持实时查看运行数据，如任务数量、调度次数、执行器数量等；以及调度报表，如调度日期分布图，调度成功分布图等；
9. 容器化：提供官方 Docker 镜像，并实时更新推送 Docker Hub，进一步实现产品开箱即用；
10. 用户管理：支持在线管理系统用户，存在管理员、普通用户两种角色；
11. 权限控制：执行器维度进行权限控制，管理员拥有全量权限，普通用户需要分配执行器权限后才允许相关操作；

### 2. elastic-job
- 基于zookeeper

## 四.大数据调度工具
- 管理hadoop工作流
1. Oozie
2. Azkaban

## 五. 实现
- 集群部署，高可用，稳定，高性能
- 不强依赖DB锁
- 界面管理、日志追查
- 报警通知

### 1.调度
#### 定时执行策略
- CRON
- 固定频率
- 固定延迟
- API
- 可靠调度

#### 调度算法
1. time.sleep
- 低效

2. 时间轮算法
时间轮是一种高效利用线程资源来进行批量化调度的一种调度模型。把大批量的调度任务全部都绑定到同一个的调度器（一个线程）上面，使用这一个调度器来进行所有任务的管理，触发以及运行，能够高效的管理各种延时任务，周期任务，通知任务等等。

每个时间轮存在着 N 个槽，两个槽之间的间隔时间固定。每走一个时间间隔，指针就向前推进一格，然后开始处理当前槽内的所有任务。指针不断循环推进，直到时间轮中不存在任何任务。

当新增调度任务时，可根据任务的调度时间和当前时间计算出具体的时间槽。为了能以时间复杂度 O(1) 的代价将任务放入指定位置，需要时间槽具有随机访问的能力，为此该部分使用循环数组实现。每一个时间槽对应的任务队列长度不确定，且只需要提供顺序访问能力，为此任务队列使用单向链表实现。

每一个时间轮都有两个必备参数，时间间隔 tickDuration 和 刻度数量 ticksPerWheel。这两个参数也很好理解，时间间隔就是指针转动的频率，刻度数量就是这个表盘内任务槽的数量，拿现实中的手表来说，tickDuration 就是 1，ticksPerWheel 是 12。

> （预计执行时间 - 时间轮启动时间）% 刻度数

常规任务指由 CRON 表达式指定定时策略的任务，这一类任务的特点是 执行频率不高。采用基于数据库轮询的策略来进行调度。

查询近期要执行的任务 -> 生成执行记录写入数据库 -> 任务推进时间轮，到点触发 -> 计算下一次调度时间并同步到数据库

#### 多级时间轮
粗粒度时间轮 + 细粒度时间轮

### 可靠调度
在本该触发任务的时刻左右，恰好遭遇 宕机/停机/长GC停顿 导致跳过，
#### WAL(Write-Ahead Logging，预写式日志)
在数据写入到数据库之前，先写入到日志中

每一个任务被调度执行时，系统都会为其生成一条记录，这条记录包含了该任务实例（任务的一次运行叫任务实例）的预期调度时间。之后，首先将该记录持久化到数据库中，只有持久化成功后，该任务才会被正式推入时间轮进行调度。

一旦这一台 server 宕机，任务没有被准时执行。其他 server 就能根据已经写入数据库中的任务实例记录将其恢复，做到可靠调度

### 秒级任务
- 执行频率高
- 对可靠性要求不高

### 无锁化调度
调度器集群

一个调度器分管一批任务

一个应用A有两个worker，有两个调度server，一开始worker都连在server1上，在一个时刻worker2请求超时，worker2再次尝试请求server2，成功，server2拿锁，拿到，数据库里的记录改为server2，server2就有了应用A的信息，而server1还没有清理不需要维护的数据，这时候server1和2都持有应用A，就有可能同时调度应用A的job和workflow。

每次推job进时间轮的时候会更新job的nextTriggerTime，这就避免了同时调度时在同一个时刻重复触发。