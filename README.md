# ELK_FileBeats_DockerCompose

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

## 默认ES跟Kibana用户名及密码
```console
username:elastic
password:changeme
```

