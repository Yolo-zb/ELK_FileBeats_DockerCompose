# ELK_FileBeats_DockerCompose

## 简介
> FileBeat + ELK的日志管理工具
具体的可自行上网查阅

## How To Start
```console
$ docker-compose build
$ docker-compose up
```
## How To Set
### filebeat配置文件
> DockerCompose
```console
filebeat:
    build:
      context: filebeat/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      #挂载的Laravel日志路径
      - type: bind
        source: ~/Documents/www/kaola-api/storage/logs
        target: /usr/share/filebeat/logs/laravel
        read_only: true
      #配置文件
      - type: bind
        source: ./filebeat/config/filebeat.yml
        target: /usr/share/filebeat/filebeat.yml
        read_only: true
      #挂载的Nginx日志路径
      - type: bind
        source: ~/Documents/nginx/logs
        target: /usr/share/filebeat/logs/nginx
        read_only: true    
    environment:
      - TZ=Asia/Shanghai
    networks:
      - elk
    depends_on:
      - elasticsearch
```

> filebeat.yml
```console
filebeat.inputs:
- type: log
  enabled: true
  #配置的路径为docker-compose上对应的laravel日志存放路径
  paths:
  - /usr/share/filebeat/logs/laravel/*.log
  #这里是对laravel日志进行的多行合并正则
  multiline.pattern: '^\[\d{4}-\d{1,2}-\d{1,2} \d{1,2}:\d{1,2}:\d{1,2}\]'
  multiline.negate: true
  multiline.match: after
  #增加logs_type字段，用于传到logstash时，可判断到是对应的日志类型
  processors:
  - add_fields:
      target: fields
      fields:
        logs_type: 'laravel_log' 
- type: log
  enabled: true
  paths:
  - /usr/share/filebeat/logs/nginx/error.log
  fields:
  document_type: nginx_error
  processors:
  - add_fields:
      target: fields
      fields:
        logs_type: 'nginx_error_log'
- type: log
  enabled: true
  paths:
  - /usr/share/filebeat/logs/nginx/access.log
  fields:
  document_type: nginx_access
  processors:
  - add_fields:
      target: fields
      fields:
        logs_type: 'nginx_access_log'
#传到logstash上
output.logstash:
  hosts: ['elk_logstash_1:5000']
#打印，可用docker logs -f ekl_filebeat_1 命令看到打印内容
#output.console:
#  pretty: true
```

###logstash配置文件
> DockerCompose
```console
logstash:
    build:
      context: logstash/
      args:
        ELK_VERSION: $ELK_VERSION
    volumes:
      #系统配置文件
      - type: bind
        source: ./logstash/config/logstash.yml
        target: /usr/share/logstash/config/logstash.yml
        read_only: true
      #接收Filebeat数据的配置文件
      - type: bind
        source: ./logstash/pipeline
        target: /usr/share/logstash/pipeline
        read_only: true
    ports:
      - "5000:5000"
      - "9600:9600"
    environment:
      - TZ=Asia/Shanghai
    networks:
      - elk
    depends_on:
      - elasticsearch
```
> ./logstash/pipeline/logstash.conf
```console
input {
    #开放端口去接收数据
	beats {
		port => "5000"
	}
}

## Add your filters / logstash plugins configuration here

filter {
  mutate {
    #存到ES时有个必要字段host，但是filebeat放了在[host][name]里，这里是将[host][name]重命名为了host
    rename => { "[host][name]" => "host" }
  }
}

output {
    #打印filebeat传过来的数据
    #可用docker logs -f ekl_logstash_1 命令看到打印内容
	stdout { codec => rubydebug }
	#这里是连接ES，根据logs_type字段来建立索引，还会根据日期来命名好的
	if([fields][logs_type]=="laravel_log"){
		elasticsearch {
			hosts => "elasticsearch:9200"
			user => "elastic"
			password => "changeme"
			index => "laravel_logs_%{+YYYY_MM_dd}"
		}
	}
	if([fields][logs_type]=="nginx_error_log"){
                elasticsearch {
                        hosts => "elasticsearch:9200"
                        user => "elastic"
                        password => "changeme"
                        index => "nginx_error_logs_%{+YYYY_MM_dd}"
                }
        }

	if([fields][logs_type]=="nginx_access_log"){
                elasticsearch {
                        hosts => "elasticsearch:9200"
                        user => "elastic"
                        password => "changeme"
                        index => "nginx_access_%{+YYYY_MM_dd}"
                }
        }
}
```
## 默认ES跟Kibana用户名及密码
```console
username:elastic
password:changeme
```

