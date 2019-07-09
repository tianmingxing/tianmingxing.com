---
title: 在ElasticSearch6.8及以上版本开启安全认证功能
date: 2019-06-20 20:01:00
tags:
- elastic search
- es安全认证
categories:
- 搜索引擎
- ElasticSearch
---

> 在6.8之前免费版本并不包含安全认证功能，之后版本有开放一些基础认证功能，对于普通用户来说是够用的。

![](/images/illustration-solutions-security-keep-data.png)

免费版本
1. TLS 功能，可对通信进行加密
1. 文件和原生 Realm，可用于创建和管理用户
1. 基于角色的访问控制，可用于控制用户对集群 API 和索引的访问权限；
1. 通过针对 Kibana Spaces 的安全功能，还可允许在 Kibana 中实现多租户。

收费版本包含更丰富的安全功能，比如：
1. 日志审计
1. IP过滤
1. LDAP、PKI和活动目录身份验证
1. 单点登录身份验证（SAML、Kerberos）
1. 基于属性的权限控制
1. 字段和文档级别安全性
1. 静态数据加密支持

需要同时在ES和kibana端开启安全认证功能，下面介绍如何操作。
<!-- more -->

#  ES端

## 启用安全模块

1. 打开ES配置文件 `$ES_PATH_CONF/elasticsearch.yml` 添加设置：`xpack.security.enabled：true`，在免费版本中此项设置是禁用的，所以需要显式打开它。
1. 非集群模式下启用单节点发现 `discovery.type: single-node`。如果你是在ES集群中配置则忽略本项，因为集群模式下要配置TLS。

启用Elasticsearch安全功能时默认情况下会启用基本身份验证，要与集群通信你必须指定用户名和密码，除非启用匿名访问，否则所有不包含用户名和密码的请求都将被拒绝。

## 为系统默认用户创建密码

Elastic Stack安全功能提供内置的用户凭据可帮助你启动运行ES，这些用户具有一组固定的权限，在设置密码之前无法进行身份验证，其中elastic用户可以用来设置所有内置的用户密码。

上面的内置用户存储在一个特殊 `.security` 索引中，该索引由Elasticsearch管理。如果禁用内置用户或其密码更改，则更改将自动反映在集群中的每个节点上。但是，如果从快照中删除或恢复索引，则已应用的任何更改都将丢失。

1. 在Elasticsearch目录中运行命令：`./bin/elasticsearch-setup-passwords interactive`，回车之后为每一个用户设置独立的密码。
1. 你需要在后续步骤中使用这些内置用户，因此务必牢记前面设置的密码！

## 在集群上配置TLS

> 如果你在操作单节点ES则可以跳过本内容。

Elastic Stack安全功能使你可以加密来自Elasticsearch集群的流量。使用传输层安全性（TLS）来保护连接，传统层安全性通常称为“SSL”。

### 为每个Elasticsearch节点生成私钥和X.509证书

1. 生成CA证书 `bin/elasticsearch-certutil ca`，将产生新文件 `elastic-stack-ca.p12`。该 `elasticsearch-certutil` 命令还会提示你输入密码以保护文件和密钥，请保留该文件的副本并记住其密码。
1. 为集群中的每个节点生成证书和私钥 `bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12`，将产生新文件 `elastic-certificates.p12`。系统还会提示你输入密码，你可以输入证书和密钥的密码，也可以按Enter键将密码留空。默认情况下 `elasticsearch-certutil` 生成没有主机名信息的证书，这意味着你可以将证书用于集群中的每个节点，另外要关闭主机名验证。
1. 将 `elastic-certificates.p12` 文件复制到每个节点上Elasticsearch配置目录中。例如，`/home/es/config/certs`。无需将 `elastic-stack-ca.p12` 文件复制到此目录。
	```bash
	mkdir config/certs
	mv elastic-certificates.p12 config/certs/
	```

### 配置集群中的每个节点以使用其签名证书标识自身并在传输层上启用TLS

1. 启用TLS并指定访问节点证书所需的信息，将以下信息添加到每个节点的 `elasticsearch.yml` 文件中：
	```yaml
	xpack.security.transport.ssl.enabled: true
	xpack.security.transport.ssl.verification_mode: certificate 
	xpack.security.transport.ssl.keystore.path: certs/elastic-certificates.p12 
	xpack.security.transport.ssl.truststore.path: certs/elastic-certificates.p12 
	```
1. 如果你在创建证书时输入了密码，那可以通过下面的方法设置。
	```bash
	bin/elasticsearch-keystore add xpack.security.transport.ssl.keystore.secure_password

	bin/elasticsearch-keystore add xpack.security.transport.ssl.truststore.secure_password
	```
1. 重启Elasticsearch

配置为使用TLS的节点无法与使用未加密网络的节点通信（反之亦然）。启用TLS后，必须重新启动所有节点才能保持群集之间的通信。

你还可以选择在[HTTP层上启用TLS](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/configuring-tls.html#tls-http)，这里就不作介绍了。

### 其它组件中启用加密（非强制）

根据自己的需要配置，不配置并不影响本次启用安全认证功能。

1. 配置[X-Pack监视以使用加密连接](https://www.elastic.co/guide/en/elastic-stack-overview/6.8/secure-monitoring.html)
1. 配置[Kibana以加密浏览器和Kibana服务器之间的通信](https://www.elastic.co/guide/en/kibana/6.8/using-kibana-with-security.html)，并通过HTTPS连接到Elasticsearch。
1. 配置[Logstash以使用TLS加密](http://www.elastic.co/guide/en/logstash/6.8/ls-security.html)。
1. 配置[Beats以使用加密连接](https://www.elastic.co/guide/en/elastic-stack-overview/6.8/beats.html)。
1. 配置[Java传输客户端以使用加密通信](https://www.elastic.co/guide/en/elastic-stack-overview/6.8/java-clients.html)。
1. 配置[Elasticsearch for Apache Hadoop](https://www.elastic.co/guide/en/elasticsearch/hadoop/6.8/security.html)以使用安全传输。

# Kibana端

启用Elasticsearch安全功能后，用户必须使用有效的用户ID和密码登录Kibana。

## 将内置用户添加到Kibana

1. 在 `kibana.yml` 文件中填写连接ES的用户凭证，上一步有为 `kibana` 用户初始化密码。
	```yaml
	elasticsearch.username: "kibana"
	elasticsearch.password: "your_password"
	```
1. 如果你不想将用户ID和密码放在kibana.yml文件中明文配置，可以将它们存储在密钥库中。运行以下命令以创建Kibana密钥库并添加配置：
	```yaml
	./bin/kibana-keystore create
	./bin/kibana-keystore add elasticsearch.username
	./bin/kibana-keystore add elasticsearch.password
	```
1. 重启kibana。

# 测试

重新访问kibana会跳转到登录页面，用上面初始化的内置账号比如elastic来登录即可。进入Manager菜单可以添加用户和角色，权限是和角色绑定的。

---
参考文献
1. https://www.elastic.co/guide/en/elastic-stack-overview/6.8/ssl-tls.html
