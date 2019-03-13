---
layout: post
title: Spark分布式计算
description: "pyspark in practice"
modified: 2017-09-01
tags: [PySpark, Spark]
categories: [PySpark, Spark]
---

spark，可谓分布式计算的屠龙宝刀，速度超快、使用超简单，关键[官方文档](https://spark.apache.org/docs/latest/)写的太详细啦（感动）。不仅支持普通的分布式计算任务，还支持实时流计算（spark-streaming）以及提供分布式机器学习算法库spark-mllib。

spark提供scala、java和python三种语言的接口，本人偏好python，所以分享下pyspark的使用心得。

## 项目结构搭建
pyspark支持交互式的访问，但只是调试的时候方便。如果要写数十个甚至上百个spark任务，总需要一个项目结构，方便运行和部署。问题有了，那就google一下，果然找到了一篇不错的经验，[Best Practices Writing Production-Grade PySpark Jobs](https://developerzen.com/best-practices-writing-production-grade-pyspark-jobs-cb688ac4d20f)。整个项目结构如下：

{% capture images %}
	/images/pyspark-project.png
{% endcapture %}
{% include gallery images=images caption="pyspark项目结构" cols=1 %}

项目结构主要分为3个目录：

* jobs文件夹，里面添加不同的spark任务，一个job新建一个文件夹，相当于python的一个模块。
* libs文件夹，存放需要用到的第三方python库。由于executor上的python环境不一定有需要的库，所以需要自己上传
* shared文件夹，存放公共的类

写完spark任务后，可以通过make build进行打包操作，Makefile部分内容如下：

{% highlight Python %}
build: clean
	mkdir ./dist
	cp ./src/main.py ./dist
	cp ./src/config.py ./dist
	cd ./src && zip -r ../dist/jobs.zip . -x main.py -x config.py -x \*libs\*
	cd ./src/libs && zip -r ../../dist/libs.zip .
	tar zcvf dist.tar.gz dist/
{% endhighlight %}

得到dist.tar.gz文件就可以上传到服务器，解压之后运行如下命令就可以执行：

{% highlight Python %}
nohup spark-submit --master yarn --py-files jobs.zip,libs.zip,config.py main.py --mode prod --job prediction --job-args date=20170926 &
{% endhighlight %}

上述命令表示集群模式运行spark任务，通过yarn进行资源分配。

## executor日志查看
在调试spark任务的时候，往往需要查看运行日志。因为spark任务是分布式运行的，由driver将任务发送到yarn分配的executor上执行，程序分为两部分，一部分是在driver上运行的，另一部分在executor上执行。driver上的日志可以在本地driver机器上看到，日志配置可以在spark的conf目录下的log4j.properties中进行配置，比如：

{% highlight Python %}
log4j.rootCategory=INFO, DRFA
log4j.appender.console=org.apache.log4j.ConsoleAppender
log4j.appender.console.target=System.err
log4j.appender.console.layout=org.apache.log4j.PatternLayout
log4j.appender.console.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
log4j.appender.DRFA=org.apache.log4j.DailyRollingFileAppender
log4j.appender.DRFA.File=log/spark.log
log4j.appender.DRFA.layout=org.apache.log4j.PatternLayout
log4j.appender.DRFA.layout.ConversionPattern=%d{yy/MM/dd HH:mm:ss} %p %c{1}: %m%n
# Settings to quiet third party logs that are too verbose
log4j.logger.org.eclipse.jetty=WARN
log4j.logger.org.eclipse.jetty.util.component.AbstractLifeCycle=ERROR
log4j.logger.org.apache.spark.repl.SparkIMain$exprTyper=INFO
log4j.logger.org.apache.spark.repl.SparkILoop$SparkILoopInterpreter=INFO
log4j.logger.org.apache.hadoop.fs.DfsInputStream=INFO
log4j.logger.org.apache.hadoop.hive.ql.io.orc.RecordReaderImpl=INFO
log4j.logger.org.apache.hadoop.hive.ql.io.*=DEBUG
log4j.logger.org.apache.spark.scheduler=ERROR
{% endhighlight %}

但是对于pyspark，如果要在executor上打印需要的日志，不能使用log4j。因为运行pyspark任务的worker是由executor上的jvm进程spawn出的子进程，只能接受命令，而不能回调（参看[pyspark运行时架构](https://cloud.tencent.com/developer/article/1005418)）。[pyspark-logging-from-the-executor](https://stackoverflow.com/questions/40806225/pyspark-logging-from-the-executor)给的一个解决方案是对于每一个python worker都设置python自己的logging模块，在shared模块中新建一个sparklogger.py，代码如下：

{% highlight Python %}
import os
import logging
import sys


class YarnLogger:
    @staticmethod
    def setup_logger():
        if not 'LOG_DIRS' in os.environ:
            sys.stderr.write('Missing LOG_DIRS environment variable, pyspark logging disabled')
            return

        file = os.environ['LOG_DIRS'].split(',')[0] + '/spark.log'
        logging.basicConfig(filename=file, level=logging.INFO,
                            format='%(asctime)s.%(msecs)03d %(levelname)s %(module)s - %(funcName)s: %(message)s')

    def __getattr__(self, key):
        return getattr(logging, key)

YarnLogger.setup_logger()
{% endhighlight %}

并在自己的job中import进来，使用实例化之后的YarnLogger，就可以像使用logging一样打印需要的日志了。

## spark配置
spark配置项很多，这里挑选几个核心的配置介绍一下：
{% highlight Python %}
# driver进程的内存，如果不需要进行大量数据汇总处理，则对内存需求不大
spark.driver.memory 8G

# 当需要连接mysql的时候，需要额外添加jar包
spark.driver.extraClassPath=/path/to/mysql-connector-java-5.1.38.jar


# executor settings
# 配置executor实例个数，就是jvm进程个数
spark.executor.instances 40

# 配置executor核数，代表了单个进程可以执行task的最大并发数，比如一个RDD分了200份，即200个partition，那么进行算子操作比如map时就是有200个task，整个集群配置的最大task并发数就是 spark.executor.instances * spark.executor.cores，那么200个task就可以达到最大并发
spark.executor.cores 5

# 配置executor内存，代表单个进程占用的内存
spark.executor.memory 8G

# 上述三项的配置需要参考集群中单个机器的配置决定的

# 任务调度模型，默认是FIFO，就是任务先进先出，但是对于任务并发的情况下不太友好。FAIR可以使得多个任务公平获取计算资源
spark.scheduler.mode FAIR

{% endhighlight %}

## spark任务
如何写spark任务，官方文档有很详细的例子，主要是理解RDD transformations（map、flatMap、mapPartitions、filter、reduceByKey等）的使用方式，这些transformations都是lazy的，只有在实施action操作（collect、count、saveAsTextFile等）的时候才会真正进行执行。

另一个需要注意的地方就是传给RDD transformations操作的函数需要是可以序列化的，因为计算操作需要从driver发送到executor上执行。如果引用了不能序列化对象，比如数据库连接等，driver会报错。

最后需要注意的是对于数据库连接初始化这种比较重的操作，可以实现成单例模式，避免重复初始化。

