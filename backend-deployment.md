---
title: 如何部署后端服务
---

[[_TOC_]]

这里描述如何安装相关软件，增加配置，并部署后端产出物的整个过程。

以下以 Ubuntu 18.04 为例，假设访问地址为 `http://dev-app.rqm.yldev.net`。

# 准备服务器
应在 VMware vSphere 创建一(或多)台虚拟机，或由项目方 IT 提供一(或多)台 Linux 服务器。
- 安装有 OpenSSH Server，用于远程访问。
- 应仅对 Web 服务器开放 18080 ~ 18100 端口

# 安装 OpenJDK 8
应使用 `apt-get install openjdk-8-jdk` 安装 OpenJDK 8。

# 安装业务服务
后端业务服务应安装为 Systemd 服务，由以下几部分配置组成。在配置完成后，应使用 `systemctl enable xxx` 设置为重启时自动启动。

## Systemd 配置
这里定义某个业务服务的 Systemd 配置，描述相关账户、加载的环境变量，以及执行命令等。
```bash
# /etc/systemd/system/rqm-prepare.service
[Unit]
Description=rqm-prepare
After=syslog.target

[Service]
User=nobody # 以非 root 身份运行
EnvironmentFile=-/etc/default/rqm-prepare # 从此文件加载环境变量
WorkingDirectory=/opt/rqm/
ExecStart=/usr/bin/java $LOGGING $JAVA_OPTS -Dserver.port=${PORT} -Djava.security.egd=file:///dev/urandom -jar $BINARY
SuccessExitStatus=143
Restart=on-failure # 如果是非正常退出则重启服务
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

## 环境变量
某个服务运行时所需的环境变量均在此文件中配置，包括运行的端口号、可执行文件以及时区等。

```bash
# /etc/default/rqm-prepare
PORT=18082
BINARY=/opt/rqm/rqm-prepare.jar
JAVA_OPTS=-Xms256m -Xmx1024m 
TZ=Asia/Shanghai # 指定当前服务运行在东八区
LOGGING="-Dlogging.config=/etc/rqm/logback.xml" # 按照指定的配置输出日志
# Service References
RETROFIT_ENDPOINTS_DOCUMENT_BASE_URL=http://127.0.0.1:18083/ # 其他服务的引用地址
```

## 日志配置
这里定义业务服务输出的日志级别及目标。

```xml
<!-- /etc/rqm/logback.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProperty scope="context" name="name" source="spring.application.name" defaultValue="localhost"/>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}:%L - %msg%n</pattern>
        </encoder>
    </appender>
    <appender name="GELF" class="biz.paluch.logging.gelf.logback.GelfLogbackAppender">
        <host>udp:graylog.ylops.net</host>
        <port>12201</port>
        <version>1.1</version>
        <extractStackTrace>true</extractStackTrace>
        <filterStackTrace>true</filterStackTrace>
        <additionalFields>name=${name}</additionalFields>
        <mdcProfiling>true</mdcProfiling>
        <timestampPattern>yyyy-MM-dd HH:mm:ss,SSSS</timestampPattern>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>INFO</level>
        </filter>
    </appender>

    <root level="DEBUG">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="GELF" />
    </root>
</configuration>
```

# 部署产出物
应从后端版本库 CI 中下载产出物，并拷贝至 `/opt/rqm/` 目录中。

# 启动并测试
可使用 `systemctl start xxx` 启动单个业务服务。使用 `curl http://localhost:18082/` 访问该服务。
