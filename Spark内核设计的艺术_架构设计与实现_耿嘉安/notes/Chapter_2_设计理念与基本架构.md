# 第二章 设计理念与基本架构

## 2.3 Spark基本设计思想

### 2.3.1 Spark模块设计

#### 1. Spark核心功能

    (1) 基础设施
        Spark配置（SparkConf）：用于管理Spark应用程序的各种配置信息；
        Spark内置的RPC框架：用于Spark各个组件间的通信；
        事件总线（ListenerBus）：SparkContext内部各个组件间使用事件——监听器模式异步调用的实现；
        度量系统：完成对整个Spark集群中各个组件运行期状态的监控。
    (2) SparkContext
    (3) SparkEnv
    (4) 存储体系
    (5) 调度系统
    (6) 计算引擎

#### 2. Spark扩展功能

    (1) Spark SQL
    (2) Spark Streaming
    (3) GraphX
    (4) MLlib

### 2.3.2 Spark模型设计

#### 1. Spark编程模型


CoarseGrainedSchedulerBackend进程内部创建Executor



#### 2. RDD计算模型




