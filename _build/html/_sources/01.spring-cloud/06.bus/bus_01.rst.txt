简介
======================================

消息总线使用轻量级消息代理实现构建一个公共的消息主题让所有微服务实例都连接上来，在这个消息主题中生产的消息所有实例都能监听和消费。

快速入门
=======================================

1.部署搭建kafka相关环境

2.修改config-server配置中心，并在pom.xml中添加如下依赖，其它需要更新外部配置文件的服务也加入该依赖::

    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bus-kafka</artifactId>
    </dependency>

3.配置文件中配置kafka相关连接信息:

4.设置配置文件仓库更新时通过Web Hook发起请求/bus/refresh?destination= 到配置中心服务    