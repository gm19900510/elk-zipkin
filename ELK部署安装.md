# 一、前言
各组件介绍请转阅：[利用ELK 搭建日志分析系统（一）—— 组件介绍](https://gaoming.blog.csdn.net/article/details/122101398)

# 二、ELK部署
> `本文基于docker进行部署`
## 2.1 镜像下载及初始化容器
### 2.1.1 创建文件夹

```sh
mkdir -p /opt/elk/elasticsearch/{conf,data}
mkdir -p /opt/elk/logstash/config
mkdir -p /opt/elk/kibana/config
```
### 2.1.2 创建初始镜像

```sh
docker run -itd --name elk sebp/elk:7.16.2
```

### 2.1.3 将配置复制到宿主机
```sh
docker cp -a elk:/opt/kibana/config/kibana.yml /opt/elk/kibana/config/
docker cp -a elk:/opt/logstash/config /opt/elk/logstash/
```
### 2.1.4 删除已创建容器

```sh
docker rm -f elk
```
## 2.2 配置文件修改
### 2.2.1 配置logstash
![在这里插入图片描述](https://img-blog.csdnimg.cn/0645738d5cd0475498e6d3b8699af02d.png)
```bash
cd /opt/elk/logstash/config
vi pipelines.yml
```
>修改`pipelines.yml`，更改配置文件加载目录如下图（`按需修改`）

![在这里插入图片描述](https://img-blog.csdnimg.cn/6afec66d0bcd445eba6e840fd73cf84f.png)
>修改`logstash-sample.conf`，更改索引名称生成规则及`input`方式及端口

![在这里插入图片描述](https://img-blog.csdnimg.cn/859eae01a4d8476a817fa9b86f32cf5c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
> 为支持`sprinboot`中`logback`将`beats`改为`tcp`，避免出现`Invalid version of beats protocol`异常
### 2.2.2 汉化kibana
在`kibana.yml`文件中加入`i18n.locale: "zh-CN"`配置以支持汉化，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/4cdc10fd1d0c4c4caf632ce976d6f497.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 2.3 创新创建容器
### 2.3.1 命令
```bash
docker run -tid -p 5601:5601 -p 5044:5044 -p 5045:5045 -p 9200:9200 -p 9300:9300 \
     -v /opt/elk/kibana/config/kibana.yml:/opt/kibana/config/kibana.yml \
     -v /opt/elk/elasticsearch/data:/var/lib/elasticsearch \
     -v /opt/elk/logstash/config:/opt/logstash/config \
     --restart=always --name elk sebp/elk:7.16.2
```

>命令解释：

-    -p 5601:5601 映射kibana端口
-    -p 9200:9200 映射es端口
-    -p 5044:5044 映射logstash端口，支持`sprinboot`中`logback`
-    -p 5045:5045 映射logstash端口，支持`filebeat`日志采集
-    -v /opt/elk/kibana/config/kibana.yml:/opt/kibana/config/kibana.yml 挂载`kibana`配置文件
-    -v /opt/elk/elasticsearch/data:/var/lib/elasticsearch 挂载`elasticsearch`数据源
-    -v /opt/elk/logstash/config:/opt/logstash/config  挂载`logstash`配置

### 2.3.2 常见问题
启动ELK时，出现以下异常：
```
RROR: [1] bootstrap checks failed. You must address the points described in the following [1] lines before starting Elasticsearch.
bootstrap check failure [1] of [1]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
ERROR: Elasticsearch did not exit normally - check the logs at /var/log/elasticsearch/elasticsearch.log
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/14a26c51988441d9835ac41be740e398.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
解决方案：
在宿主机`/etc/sysctl.conf`中新增`vm.max_map_count=262144`　
```bash
vi /etc/sysctl.conf
vm.max_map_count=262144　
```
加载参数

```bash
sysctl -p
```
或在宿主机单独执行以下命令：

```bash
sysctl -w vm.max_map_count=262144
```
# 三、springboot集成logstash
## 2.1 新增maven依赖

```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>7.0.1</version>
</dependency>
```
## 2.2 logback-spring.xml配置
在`XML`中新增`LOGSTASH`配置，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/280a412b49f74a978db468a55a67bec5.png)

```xml
    <!-- 这里是你的业务的包名 -->
    <logger name="cn.net.hylink">
		<level value="DEBUG" />
	</logger>
	<!-- logstash ip和暴露的端口，我目前理解就是通过这个地址把日志发送过去  -->
	<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
		<!-- 和logstash 的input 配置的端口保持一致 -->
        <destination>192.168.3.202:5044</destination> 
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>
```

```xml
    <!-- 开发环境:打印控制台-->
    <springProfile name="dev">
        <root level="info">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="LOGSTASH" />
        </root>
    </springProfile>
```
完整`logback-spring.xml`如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- 日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出 -->
<!-- scan:当此属性设置为true时，配置文档如果发生改变，将会被重新加载，默认值为true -->
<!-- scanPeriod:设置监测配置文档是否有修改的时间间隔，如果没有给出时间单位，默认单位是毫秒。
                 当scan为true时，此属性生效。默认的时间间隔为1分钟。 -->
<!-- debug:当此属性设置为true时，将打印出logback内部日志信息，实时查看logback运行状态。默认值为false。 -->
<configuration scan="true" scanPeriod="10 seconds">
    <contextName>logback</contextName>

    <!-- name的值是变量的名称，value的值时变量定义的值。通过定义的值会被插入到logger上下文中。定义后，可以使“${}”来使用变量。 -->
    <property name="log.path" value="logs"/>
    <property name="service.name" value="hlk_qingzhi_service"/>

    <!--0. 日志格式和颜色渲染 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>
    <!-- 彩色日志格式 -->
    <property name="CONSOLE_LOG_PATTERN"
              value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

    <!--1. 输出到控制台-->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <!--此日志appender是为开发使用，只配置最底级别，控制台输出的日志级别是大于或等于此级别的日志信息-->
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>debug</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <!-- 设置字符集 -->
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="LOGFILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/${service.name}.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志归档 -->
            <fileNamePattern>${log.path}/${service.name}-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!--<file class="ch.qos.logback.classic.filter.LevelFilter">
        </file>-->
    </appender>

    <!--2. 输出到文档-->
    <!-- 2.1 level为 DEBUG 日志，时间滚动输出  -->
    <appender name="DEBUG_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/${service.name}-DEBUG.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志归档 -->
            <fileNamePattern>${log.path}/web-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录debug级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>debug</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 2.2 level为 INFO 日志，时间滚动输出  -->
    <appender name="INFO_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/${service.name}-INFO.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${log.path}/${service.name}-INFO-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录info级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>info</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 2.3 level为 WARN 日志，时间滚动输出  -->
    <appender name="WARN_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/${service.name}-WARN.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/${service.name}-WARN-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录warn级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>warn</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>

    <!-- 2.4 level为 ERROR 日志，时间滚动输出  -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 正在记录的日志文档的路径及文档名 -->
        <file>${log.path}/${service.name}-ERROR.log</file>
        <!--日志文档输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${log.path}/${service.name}-ERROR-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文档保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文档只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    </appender>
    
    <!-- 这里是你的业务的包名 -->
    <logger name="cn.net.hylink">
		<level value="DEBUG" />
	</logger>
	<!-- logstash ip和暴露的端口，我目前理解就是通过这个地址把日志发送过去  -->
	<appender name="LOGSTASH" class="net.logstash.logback.appender.LogstashTcpSocketAppender">
		<!-- 和logstash 的input 配置的端口保持一致 -->
        <destination>192.168.3.202:5044</destination> 
        <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder" />
    </appender>

    <!--
        <logger>用来设置某一个包或者具体的某一个类的日志打印级别、
        以及指定<appender>。<logger>仅有一个name属性，
        一个可选的level和一个可选的addtivity属性。
        name:用来指定受此logger约束的某一个包或者具体的某一个类。
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
              还有一个特俗值INHERITED或者同义词NULL，代表强制执行上级的级别。
              如果未设置此属性，那么当前logger将会继承上级的级别。
        addtivity:是否向上级logger传递打印信息。默认是true。
        <logger name="org.springframework.web" level="info"/>
        <logger name="org.springframework.scheduling.annotation.ScheduledAnnotationBeanPostProcessor" level="INFO"/>
    -->

    <!--
        使用mybatis的时候，sql语句是debug下才会打印，而这里我们只配置了info，所以想要查看sql语句的话，有以下两种操作：
        第一种把<root level="info">改成<root level="DEBUG">这样就会打印sql，不过这样日志那边会出现很多其他消息
        第二种就是单独给dao下目录配置debug模式，代码如下，这样配置sql语句会打印，其他还是正常info级别：
        【logging.level.org.mybatis=debug logging.level.dao=debug】
     -->

    <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
        不能设置为INHERITED或者同义词NULL。默认是DEBUG
        可以包含零个或多个元素，标识这个appender将会添加到这个logger。
    -->

    <!-- 开发环境:打印控制台-->
    <springProfile name="dev">
        <root level="info">
            <appender-ref ref="CONSOLE"/>
            <appender-ref ref="LOGSTASH" />
        </root>
    </springProfile>


    <!-- 测试环境:输出到文档-->
    <springProfile name="test">
        <root level="info">
        	<appender-ref ref="CONSOLE"/>
            <appender-ref ref="LOGFILE"/>
            <appender-ref ref="DEBUG_FILE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="WARN_FILE"/>
        </root>
    </springProfile>

    <!-- 生产环境:输出到文档-->
    <springProfile name="prod">
        <root level="info">
            <appender-ref ref="LOGFILE"/>
            <appender-ref ref="INFO_FILE"/>
            <appender-ref ref="ERROR_FILE"/>
            <appender-ref ref="WARN_FILE"/>
        </root>
    </springProfile>

</configuration>

```
# 四、kibana配置使用
成功抓取的日志以后进入到`kibana`当中创建索引模式，如果未抓取到日志，看下`docker logs` 是否有错误。通常是配置文件语法格式错误。

## 4.1 索引模式配置

浏览器访问：[http://192.168.3.202:5601/](http://192.168.3.202:5601/)，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/3abd92c2e5ef462dadbb2b277dbc157e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
点击管理空间，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/1de4f03dfa234dc9b0469be8a145c370.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
创建索引模式，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/8f94530c0cbd42fab043af3b8490d771.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/38f0928f9cbc43a38a57a5e39aeaab1a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
新增名称：`opt-*`，时间戳字段选择：`@timestamp`，点击`创建索引模式`，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/78e7971ccf8140bb8f3337e2a872a264.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
![在这里插入图片描述](https://img-blog.csdnimg.cn/3dd3195d2a8846aca5c535dfb79ffa32.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
## 4.2 日志检索
进入`Discover`，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/d677cceaa0ea4295ad957c388b3ce6e4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
选择索引
![在这里插入图片描述](https://img-blog.csdnimg.cn/f6e76e8619e4494c87e644dd78356f5a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
kibana的简单使用就先介绍到这里。

# 五、filebeat集成
## 5.1 配置logstash支持filebeat
在`/opt/elk/logstash/config`目录下新建`logstash-filebeat.conf`文件

```bash
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.

input {
  beats {
    port => 5045
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "filebeat-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
}
```
> `注意端口必须与宿主机进行映射及索引名称生成规则`
## 5.2 filebeat下载及配置
下载地址：[https://www.elastic.co/cn/downloads/beats/filebeat](https://www.elastic.co/cn/downloads/beats/filebeat)
> 注意`filebeat`版本必须与`logstash`版本对应

我下载的是`filebeat-7.16.2-linux-x86_64.tar.gz`，解压

```bash
tar -zxvf filebeat-7.16.2-linux-x86_64.tar.gz
```
修改`filebeat`配置

```bash
cd /opt/elk/filebeat-7.16.2-linux-x86_64
vi filebeat.yml
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/06f602b77ccd4aa5ace5d5b79c81a5bc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
>注意将`enabled`改为`true`，不然会存在不收集日志问题

![在这里插入图片描述](https://img-blog.csdnimg.cn/8489c562f26b41d1937f29139561c491.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
启动`filebeat`

```bash
cd /opt/elk/filebeat-7.16.2-linux-x86_64
./filebeat -e -c filebeat.yml
```
后台方式启动
```bash
nohup ./filebeat -e -c filebeat.yml &
```
查找已存在的`filebeat`进程
```bash
ps -ef | grep filebeat
```
`kibana`效果
![在这里插入图片描述](https://img-blog.csdnimg.cn/6f8de8b5ac974e199c89c82815b67756.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

问题参考：[https://elk-docker.readthedocs.io/](https://elk-docker.readthedocs.io/)
