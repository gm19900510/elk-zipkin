# 一、Zipkin 简介
`Zipkin` 是一个开放源代码分布式的跟踪系统，每个服务向zipkin报告计时数据，zipkin会根据调用关系通过Zipkin UI生成依赖关系图。

`Zipkin`提供了可插拔数据存储方式：`In-Memory`、`MySql`、`Cassandra`以及`Elasticsearch`。为了方便在开发环境我直接采用了`In-Memory`方式进行存储，生产数据量大的情况则推荐使用`Elasticsearch`。

# 二、Zipkin 基本术语
- Span

基本工作单元，例如，在一个新建的span中发送一个RPC等同于发送一个回应请求给RPC，span通过一个64位ID唯一标识，trace以另一个64位ID表示，span还有其他数据信息，比如摘要、时间戳事件、关键值注释(tags)、span的ID、以及进度ID(通常是IP地址)
span在不断的启动和停止，同时记录了时间信息，当你创建了一个span，你必须在未来的某个时刻停止它。

- Trace

一系列spans组成的一个树状结构，例如，如果你正在跑一个分布式大数据工程，你可能需要创建一个trace。

- Annotation

用来及时记录一个事件的存在，一些核心annotations用来定义一个请求的开始和结束
cs - Client Sent -客户端发起一个请求，这个annotion描述了这个span的开始
sr - Server Received -服务端获得请求并准备开始处理它，如果将其sr减去cs时间戳便可得到网络延迟
ss - Server Sent -注解表明请求处理的完成(当请求返回客户端)，如果ss减去sr时间戳便可得到服务端需要的处理请求时间
cr - Client Received -表明span的结束，客户端成功接收到服务端的回复，如果cr减去cs时间戳便可得到客户端从服务端获取回复的所有所需时间
![在这里插入图片描述](https://img-blog.csdnimg.cn/4c6260831768408f85896c1398ca1f36.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

# 三、Zipkin 服务安装
本文基于`docker`进行安装，存储采用`In-Memory`

```powershell
docker run -d --restart always -p 9411:9411 --name zipkin openzipkin/zipkin
```
# 四、与项目集成
## 4.1 Maven新增依赖
   

```xml
<dependency>
   <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
    <version>2.2.8.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>


<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Hoxton.SR9</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

    </dependencies>
</dependencyManagement>
```
## 4.2 修改配置文件bootstrap.yml

```yaml
  zipkin:
    base-url: http://127.0.0.1:9411/ #zipkin地址
    discovery-client-enabled: false  #不用开启服务发现
  sleuth:
    sampler:
      probability: 1.0 #采样百分比
```
## 4.3 修改logback.xml
主要添加这个打印格式：
```xml
<property name="log.pattern" value="%d{HH:mm:ss.SSS} [%X{traceId}/%X{spanId}] [%thread] %-5level %logger{20} - [%method,%line] - %msg%n" />
```
日志效果：

```powershell
19:24:22.651 [6916352b21b8a493/68a26f7faac91083] [http-nio-9201-exec-6] INFO  c.r.s.c.SysUserController - [getInfo,162] - 接收参数：1
```
# 五、Zipkin整体效果
依赖关系图
![在这里插入图片描述](https://img-blog.csdnimg.cn/d7cecfb603f941deb96b53b54195e27d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
链路追踪
![在这里插入图片描述](https://img-blog.csdnimg.cn/702f906ea9644e3e9f90b440467d64c4.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
# 六、其他链路追踪选型SkyWalking

![在这里插入图片描述](https://img-blog.csdnimg.cn/94b44c61cbd84ffb9309ca1684996884.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
参考：
[https://www.jianshu.com/p/0fbbf99a236e](https://www.jianshu.com/p/0fbbf99a236e)
[https://blog.csdn.net/qq_20630341/article/details/120530263](https://blog.csdn.net/qq_20630341/article/details/120530263)

