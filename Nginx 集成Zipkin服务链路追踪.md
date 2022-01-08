# 一、背景
`Nginx`是一款自由的、开源的、高性能的HTTP服务器和反向代理服务器，对其进行跟踪可以帮助我们更好的了解应用服务的运行状况。本文将详细介绍`Nginx`的链路跟踪的安装过程。

# 二、操作步骤
> 本文基于`docker`构建集成镜像，本地化安装方式请参照本文后续参照链接

载Dockerfile并编译部署

[下载链接](https://download.csdn.net/download/ctwy291314/74884722)

`nginx-zipkin-docker`目录结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/bade14b3d353400a945c7141462e5626.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
解压并构建镜像
```shell
tar -xzvf nginx-zipkin-docker.tgz
cd nginx-zipkin-docker
// 编译docker
docker build --rm --tag nginx-zipkin:1.21.5 .
```
通过修改`nginx.conf`更改默认反向代理地址，默认如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/37ec2cf1c0a543dc934c2fca0c2daa82.png)
完整`nginx.conf`如下：

```css
load_module modules/ngx_http_opentracing_module.so;

user  nginx;
worker_processes  auto;

#错误日志也应该指定到咱们编译安装的nginx目录中
error_log  /nginx/logs/error.log notice;

pid        /var/run/nginx.pid;

#Docker最终运行Nginx建议大家把后台进程关闭,默认是"on".
daemon off;

events {
    worker_connections  1024;
}


http {
  opentracing on;

  opentracing_load_tracer /usr/local/lib/linux-amd64-libzipkin_opentracing_plugin.so /etc/zipkin-config.json;

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
					  
  access_log  /nginx/logs/access.log  main;

  sendfile            on;
  keepalive_timeout   65;
  include       mime.types;
  default_type  text/html;
  charset utf-8;
	
  server {

    listen       80 default_server;
    listen       [::]:80 default_server;
    server_name  _;

    #别忘记修改站点代码的根目录也要指定到咱们编译安装nginx的目录中哟~
	root         /nginx/html;
	
	location /test/ {
		opentracing_operation_name $uri;
		opentracing_tag nginx.upstream_addr $upstream_addr;
		opentracing_tag _nginxIp hostIp;
		opentracing_trace_locations off;
		proxy_pass http://192.168.2.32:70/;
		opentracing_propagate_context;
	}
	
	error_page 404 /404.html;             
	location = /40x.html {
	}
	error_page 500 502 503 504 /50x.html;
	location = /50x.html {
	}
  }
}

```

运行`docker`

```shell
docker run --name nginx-zipkin -p 80:80 -e "COLLECTOR_HOST=${ZIPKIN_ENDPOINT}?" -d nginx-zipkin:1.21.5
# ${ZIPKIN_ENDPOINT}是准备工作中保存的v1版本的Agent接入点信息。不包括“http://”部分，且以英文问号（?）结尾。
```

例如：

```bash
docker run --name nginx-zipkin  -p 90:80 -e "COLLECTOR_HOST=192.168.2.32:9411/api/v1/spans?" -d nginx-zipkin:1.21.5
```
`linux-amd64-nginx-1.21.5-ngx_http_module.so.tgz`下载地址：

[https://github.com/opentracing-contrib/nginx-opentracing/releases/](https://github.com/opentracing-contrib/nginx-opentracing/releases/)

在`/etc/zipkin-config.json`文件中配置`Zipkin`参数

```bash
{
  "service_name": "nginx",
  "collector_host": "zipkin"
}
```
通过`sample_rate`配置采样比例

```bash
// 10%的采样比例
"sample_rate":0.1
```
`linux-amd64-libzipkin_opentracing_plugin.so` 下载地址：

[https://github.com/rnburn/zipkin-cpp-opentracing/releases/](https://github.com/rnburn/zipkin-cpp-opentracing/releases/)

`nginx` 下载地址：

[http://nginx.org/en/download.html](http://nginx.org/en/download.html)

# 三、效果

浏览器访问：[http://127.0.0.1:90/test/1.txt](http://127.0.0.1:90/test/1.txt) ，`zipkin` 展示效果如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/958be43490a54c2baa8c20c10de4f09c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)
# 四、 其他参考
在物理机上部署可参考：[https://help.aliyun.com/document_detail/196710.html](https://help.aliyun.com/document_detail/196710.html)

使用`Skywalking`对`Nginx`进行链路追踪可参考：[https://help.aliyun.com/document_detail/197661.html](https://help.aliyun.com/document_detail/197661.html)
