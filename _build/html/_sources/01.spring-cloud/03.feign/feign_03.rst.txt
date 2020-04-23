Feign 简介
======================================

Spring Cloud Feign整合了Spring Cloud Ribbon与Spring Cloud Hystrix，除提供两者强大功能外，提供了一种声明式的web服务客户端定义方式。只用使用注解便能轻松使用Feign客户端。

快速入门
--------------------------------------

1.创建一个Spring boot项目，命名为hello-service，在pom.xml中添加Hystrix依赖::
  
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-feign</artifactId>
    </dependency>

2.在主类中使用@EnableFeignClients注解开启Feign功能支持::

    @EnableFeignClients
    @EnableDiscoveryClient
    @SpringBooApplication
    public class ConsumerApplication{

        public static void main(String[] args){

            SpringApplication.run(ConsumerApplication.class, args);

        }

    }

3.定义SslmRemoteService接口，通过@FeignClient的value属性指定服务名来绑定服务，fallback属性指定服务降级调用方法类，path属性指定公共的URL::

    @FeignClient(value = "srm-supplier-lifecycle", fallback = SslmRemoteServiceFallback.class, path = "/v1/{organizationId}")
    public interface SslmRemoteService{

        /**
        *创建调查表头并发布调查表
        *
        * @param organizationId
        * @param investigateHeaderList
        * @return List<InvestigateHeaderVO>
        */
        @PostMapping("/investigate/save-release")
        List<InvestigateHeaderVO> batchSaveAndReleaseInvestgHeader(@PathVariable("organizationId") Long organizationId,
                                                                    @RequestBody List<InvestigateHeaderVO> investigateHeaderList);


    }

4.定义SslmRemoteServiceFallback回调方法类::

    @Component
    @SuppressWarnings("all")
    public class SslmRemoteServiceFallback implements SslmRemoteService{

        private static final Logger LOGGER = LoggerFactory.getLogger(SslmRemoteServiceFallback.class);

        @Override
        public List<InvestigateHeaderVO> batchSaveAndReleaseInvestgHeader(Long organizationId, List<InvestigateHeaderVO> investigateHeaderList) {
            LOGGER.error("Post batch save and release investgHeader fail");
            throw new CommonException(BaseConstants.ErrorCode.ERROR);
        }
    }

Feign 属性配置
======================================

Spring Cloud Feign整合了Spring Cloud Ribbon和Spring Cloud Hystrix，所以对于负载均衡、熔断降级的配置可以按Ribbon和Hystrix的配置。

Ribbon配置
--------------------------------------

@FeignClient中name或value属性指定的服务名即Ribbon客户端的名，因此Feign中想对不同客户端做不同配置可以根据<client>.ribbon.key=value进行配置

Hystrix配置
---------------------------------------

    feign.hystrix.enabled: 设置是否开启Feign 客户端Hystrix支持。

禁用Hystrix配置
---------------------------------------

除通过feign.hystrix.enabled设置外还可针对服务客户端关闭Hystrix支持

1.创建关闭Hystrix配置类::

    @Configuration
    public class DisableHystrixConfiguration{
        
        @Bean
        @Scope("prototype")
        public Feign.Builder feignBuilder(){
            return Feign.builder();
        }

    }

2.通过@FeignClient注解中，configuration参数引入配置::

    @FeignClient(value = "hello-service", configuration = DisableHystrixConfiguration.class)
    public interface HelloService{
        
    }

Feign配置属性
-----------------------------------------

    feign.compression.request.enabled: 设置是否开启请求GZIP压缩功能
    feign.compression.response.enabled: 设置是否开启响应GZIP压缩功能
    feign.compression.request.mime-type: 设置GZIP压缩请求的类型，默认text/xml,application/xml,application/json
    feign.compression.request.min-request-size: 设置GZIP压缩请求大小下限，超过的请求将被压缩