搭建服务注册中心
======================================

1.创建基础Spring Boot工程，命名为eureka-server，并在pom.xml中引入必要的依赖::

        <parent>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-parent</artifactId>
          <version>1.5.14.RELEASE</version>
          <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <dependencies>
          <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
          </dependency>
          <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
              <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
              </exclusion>
             </exclusions>
           </dependency>
           <dependency>
             <groupId>org.springframework.cloud</groupId>
             <artifactId>spring-cloud-starter-eureka-server</artifactId>
           </dependency>
        </dependencies>        

2.@EnableEurekaServer注解启动一个服务注册中心::

        @EnableEurekaServer
        @SpringBootApplication
        public class EurekaServerApplication {

                public static void main(String[] args) {

                        SpringApplication.run(EurekaServerApplication.class, args);

                }

        }

3.添加bootstrap.yum文件::

        server:
          port: 8000
        spring:
          application:
            name: eureka-server

4.添加application.yml配置文件::

        eureka:
          instance:
            preferIpAddress: true
            leaseRenewalIntervalInSeconds: 10
            leaseExpirationDurationInSeconds: 30
          client:
            register-with-eureka: false
            fetch-registry: false
            serviceUrl:
              defaultZone: ${EUREKA_DEFAULT_ZONE:http://localhost:8000/eureka/}
            registryFetchIntervalSeconds: 10
            disable-delta: true
          server:
            evictionIntervalTimerInMs: 4000

eureka.client.register-with-eureka: 设置是否向注册中心注册服务，因为本身为注册中心不需要注册自己。如果是注册中心集群则不需要此配置，默认为true。

eureka.client.fetch-registry: 设置是否去注册中心检索服务，因为本身为注册中心维护有服务列表不用去检索。如果为注册中心集群则不需此配置，默认为true。

eureka.client.serviceUrl.defaultZone: 注册中心地址

eureka.client.registryFetchIntervalSeconds: 间隔多久去拉取服务注册信息，默认30秒

服务提供者
======================================

1.创建一个Spring Boot项目，命名为oauth-server，pom.xml如下::
 
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-undertow</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.springframework.boot</groupId>
                    <artifactId>spring-boot-starter-tomcat</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

2.添加@EnableDiscoveryClent注解，开启服务发现功能::
        
        @EnableEurekaClient
        @SpringBootApplication
        public class OauthServerApplication {

                public static void main(String[] args) {

                        SpringApplication.run(OauthServerApplication.class, args);

                }
        }

3.添加bootstrap.yml配置文件::

        server:
          port: 8020
        spring:
          application:
            name: oauth-server

4.添加applocation.yml文件::
    
        eureka:
          client:
            serviceUrl:
              defaultZone: ${EUREKA_DEFAULT_ZONE:http://localhost:8000/eureka/}
            registryFetchIntervalSeconds: 10
            
        

