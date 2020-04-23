Hystrix 简介
=====================================

Spring Cloud Hystrix具备服务降级、服务熔断、线程和信号隔离、请求缓存、请求合并、服务监控等功能。在微服务系统中存在多个服务单元，服务单元之间存在广泛依赖关系，当一个服务节点发生故障时可能会引发故障的蔓延导致整个系统不可用。在分布式架构中，断路器模式的作用就是为了解决这种问题。当某个单元发生故障是，熔断监控器将向调用方返回一个错误响应，防止线程因故障长时间占用得不到释放，导致故障蔓延系统瘫痪。

快速入门
--------------------------------------

1.创建一个Spring boot项目，命名为hello-service，在pom.xml中添加Hystrix依赖::
  
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-hystrix</artifactId>
    </dependency>

2.在主类中使用@EnableCircuitBreaker注解开启断路器功能::

    @EnableCircuitBreaker
    @EnableDiscoveryClient
    @SpringBooApplication
    public class ConsumerApplication{

        @Bean
        @LoadBalanced
        RestTemplate restTemplate(){

            return new RestTemplate;

        }

        public static void main(String[] args){

            SpringApplication.run(ConsumerApplication.class, args);

        }
    }

3.添加服务方法,@HystrixCommand启动熔断功能，fallbackMethod指定服务降级调用方法，当调用服务发生故障或长时间未响应将调用服务降级方法，做后续处理::

    @Service
    public class HelloService{

        @Autowired
        RestTemplate restTemplate;

        @HystrixCommand(fallbackMethod = "helloFallback")
        public String helloService(){

            return restTemplate.getForEntity("http://HELLO-SERVICE/hello", String.class).getBody();
            
        }

        public String helloFallback(){

            return "error";
        }

    }

属性详情
--------------------------------------

属性配置优先级

    全局默认值: 没有显示设置属性值，Hystrix默认属性值

    全局配置属性: 在配置文件中显示指定属性值覆盖默认值。系统启动时或通过Spring Cloud Config和Spring Cloud Bus动态刷新配置功能配合下，实现对系统默认值覆盖。

    实例默认值: 通过代码对实例设置属性值覆盖默认值

    实例配置属性: 通过配置文件来为实例进行属性配置

常用配置属性
----------------------------------------

hystrix.command.default.execution.isolation.strategy: 配置执行隔离策略，THREAD通过线程池隔离，默认值。SEMAPHORE通过信号量隔离。

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds: 配置执行超时时间，单位毫秒。调用接口超过设置时间将触发服务降级。

hystrix.command.default.execution.timeout.enabled: 设置是否启用超时时间，默认是true，如果设置成false，则超时时间配置不起作用。

hystrix.command.default.fallback.enabled: 是否启用服务降级策略，默认为true。

hystrix.threadpool.default.coreSize: 执行命令线程池的核心线程数，即命令执行的最大并发数。默认值为10。

hystrix.threadpool.default.maximumSize: 最大执行线程数。

hystrix.threadpool.default.maxQueueSize: 设置线程池最大队列大小。当为-1使用SynchronousQueue实现的队列，否则使用LinekedBlockingQueue实现的队列。