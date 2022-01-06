# 一、Kibana 安全认证
## 1.1 架构方案一
通过`nginx`反向代理`kibana`，开启`Kibana` 登录认证。

>本机基于`docker`部署`nginx`

```bash
# 启动容器
docker run --name nginx -p 8080:80 -d nginx

#查看运行容器的ID
docker ps
 
#进入nginx容器
docker exec -it nginx /bin/bash
 
#容器内部操作
#更新软件源
apt-get update
  
#安装apache2-utils
apt-get install apache2-utils

# 新建文件夹 
mkdir -p /etc/nginx/db/
 
#创建用户名，kibana为用户名
htpasswd -cm /etc/nginx/db/passwd.db kibana
 
#输入密码（自动弹出）
New password: 
Re-type new password: 
 
#查看用户和密码
cat /etc/nginx/db/passwd.db
 
#退出
exit

#拷贝出配置文件default.conf
docker cp nginx:/etc/nginx/conf.d/default.conf /opt/nginx/
```


修改`nginx`配置文件`default.conf`

```shell
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
		auth_basic "secret";
		auth_basic_user_file /etc/nginx/db/passwd.db;
		proxy_pass http://172.19.34.205:5601/;
		proxy_set_header Host $host:$server_port;
		proxy_set_header X-Real_IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Scheme $scheme;
		proxy_connect_timeout 3;
		proxy_read_timeout 3;
		proxy_send_timeout 3;
		access_log off;
		break;

    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
}
```

`nginx`配置文件`default.conf`考入至容器

```shell
docker cp /opt/nginx/default.conf nginx:/etc/nginx/conf.d/
```
重启容器

```shell
docker restart nginx
```

文章参考：[http://technosophos.com/2014/03/19/ssl-password-protection-for-kibana.html](http://technosophos.com/2014/03/19/ssl-password-protection-for-kibana.html)

## 1.2 架构方案二
通过启用`xpack`插件，开启`Kibana`登录认证。

结合第二部分，修改`kibana.yml`配置文件，新增以下内容：

```bash
elasticsearch.username: "kibana_system"
elasticsearch.password: "之前设置的密码"
```

>**登录时请使用：`elastic/之前设置的密码` 进行登录**，登录`Kibana`用`elastic`用户，使用超级用户来管理空间、创建用户、和分配角色。

![在这里插入图片描述](https://img-blog.csdnimg.cn/406451f832004702b2dea98d0d5b5406.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZ21IYXBweQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

# 二、Elasticsearch 安全认证
通过启用`xpack`插件，开启`Elasticsearch` 登录认证。

```bash

# Elasticsearch 配置文件拷出
docker cp elk:/etc/elasticsearch/elasticsearch.yml /opt/elk/elasticsearch/conf/

#在elasticsearch.yml文件中新增以下内容
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true

#Elasticsearch 配置文件拷入至容器
docker cp /opt/elk/elasticsearch/conf/elasticsearch.yml elk:/etc/elasticsearch/

#重启ElasticSearch
docker restart elk

# 进入容器
docker exec -it elk /bin/bash

#完成内置用户密码的设置
root@85f62c9ea7c2:/# ./opt/elasticsearch/bin/elasticsearch-setup-passwords interactive
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,kibana_system,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]: 
Reenter password for [elastic]: 
Enter password for [apm_system]: 
Reenter password for [apm_system]: 
Enter password for [kibana_system]: 
Reenter password for [kibana_system]: 
Enter password for [logstash_system]: 
Reenter password for [logstash_system]: 
Enter password for [beats_system]: 
Reenter password for [beats_system]: 
Enter password for [remote_monitoring_user]: 
Reenter password for [remote_monitoring_user]: 
Changed password for user [apm_system]
Changed password for user [kibana_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]

```
内置用户

用户名	 | 作用
|:-------- |:-----|
elastic	|超级用户
kibana_system|用于负责Kibana连接Elasticsearch
logstash_system	|Logstash将监控信息存储在Elasticsearch中时使用
beats_system	|Beats在Elasticsearch中存储监视信息时使用
apm_system	|APM服务器在Elasticsearch中存储监视信息时使用
remote_monitoring_user	|Metricbeat用户在Elasticsearch中收集和存储监视信息时使用


为`elastic`用户设置密码后，引导密码将不再有效。并且再次执行`elasticsearch-setup-passwords`命令会抛出异常

```bash
Failed to authenticate user 'elastic' against http://***.***.***.***:9200/_security/_authenticate?pretty
Possible causes include:
 * The password for the 'elastic' user has already been changed on this cluster
 * Your elasticsearch node is running against a different keystore
   This tool used the keystore at /usr/local/es-cluster/elasticsearch-7.2.0-a/config/elasticsearch.keystore

ERROR: Failed to verify bootstrap password
```

如果忘记密码，可以先取消认证，即注释掉上面`elasticsearch.yml`中添加的两个配置，然后重启`ElasticSearch`，然后找到一个类型`.security-X`的`index`，删除掉就可以回到最初无密码认证的状态了　　

```bash
# 查看.security-X存在与否
curl http://localhost:9200/_cat/indices | grep ".security"
# 删除index，我这里是.security-7
curl -XDELETE http://localhost:9200/.security-7
```

文章参考：[https://www.elastic.co/guide/en/elasticsearch/reference/7.16/security-minimal-setup.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.16/security-minimal-setup.html)

# 三、Logstash 支持Elasticsearch 安全认证
`logstash-filebeat.conf`变更为以下内容：

```bash
# Sample Logstash configuration for creating a simple
# Beats -> Logstash -> Elasticsearch pipeline.

input {
  tcp {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "opt-%{+YYYY.MM.dd}"
    user => "logstash_system"
    password => "之前设置的密码"
  }
}
```
`logstash-filebeat.conf`变更为以下内容：

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
    user => "logstash_system"
    password => "之前设置的密码"
  }
}

```

