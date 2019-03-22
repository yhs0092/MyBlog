# 详解契约校验失败导致的微服务实例注册失败问题

相信不少同学在使用 CSEJavaSDK / ServiceComb-Java-Chassis 开发微服务的过程中都碰到过修改了服务接口后服务就无法启动的情况。控制台上会打出如下的异常栈：
```
Exception in thread "main" java.lang.IllegalStateException: ServiceComb init failed.
	at org.apache.servicecomb.core.SCBEngine.init(SCBEngine.java:220)
	at org.apache.servicecomb.core.CseApplicationListener.onApplicationEvent(CseApplicationListener.java:81)
	at org.springframework.context.event.SimpleApplicationEventMulticaster.doInvokeListener(SimpleApplicationEventMulticaster.java:172)
	at org.springframework.context.event.SimpleApplicationEventMulticaster.invokeListener(SimpleApplicationEventMulticaster.java:165)
	at org.springframework.context.event.SimpleApplicationEventMulticaster.multicastEvent(SimpleApplicationEventMulticaster.java:139)
	at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:393)
	at org.springframework.context.support.AbstractApplicationContext.publishEvent(AbstractApplicationContext.java:347)
	at org.springframework.context.support.AbstractApplicationContext.finishRefresh(AbstractApplicationContext.java:883)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:546)
	at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:139)
	at org.springframework.context.support.ClassPathXmlApplicationContext.<init>(ClassPathXmlApplicationContext.java:93)
	at org.apache.servicecomb.foundation.common.utils.BeanUtils.init(BeanUtils.java:49)
	at org.apache.servicecomb.foundation.common.utils.BeanUtils.init(BeanUtils.java:42)
	at microservice.demo.training21days.provider.AppMain.main(AppMain.java:9)
Caused by: java.lang.IllegalStateException: The schema(id=[hello]) content held by this instance and the service center is different. You need to increment microservice version before deploying. Or you can configure service_description.environment=development to work in development environment and ignore this error
	at org.apache.servicecomb.serviceregistry.task.MicroserviceRegisterTask.compareAndReRegisterSchema(MicroserviceRegisterTask.java:277)
	at org.apache.servicecomb.serviceregistry.task.MicroserviceRegisterTask.registerSchema(MicroserviceRegisterTask.java:206)
	at org.apache.servicecomb.serviceregistry.task.MicroserviceRegisterTask.registerSchemas(MicroserviceRegisterTask.java:170)
	at org.apache.servicecomb.serviceregistry.task.MicroserviceRegisterTask.doRegister(MicroserviceRegisterTask.java:122)
	at org.apache.servicecomb.serviceregistry.task.AbstractRegisterTask.doRun(AbstractRegisterTask.java:41)
	at org.apache.servicecomb.serviceregistry.task.AbstractTask.run(AbstractTask.java:53)
	at org.apache.servicecomb.serviceregistry.task.CompositeTask.run(CompositeTask.java:35)
	at org.apache.servicecomb.serviceregistry.task.ServiceCenterTask.init(ServiceCenterTask.java:82)
	at org.apache.servicecomb.serviceregistry.registry.AbstractServiceRegistry.run(AbstractServiceRegistry.java:178)
	at org.apache.servicecomb.serviceregistry.registry.RemoteServiceRegistry.run(RemoteServiceRegistry.java:86)
	at org.apache.servicecomb.serviceregistry.RegistryUtils.run(RegistryUtils.java:70)
	at org.apache.servicecomb.core.SCBEngine.doInit(SCBEngine.java:255)
	at org.apache.servicecomb.core.SCBEngine.init(SCBEngine.java:209)
	... 13 more
```

其实这里抛出的异常已经告诉大家问题的原因和解决方式了。问题原因是这个服务在服务中心里注册的契约内容和这个实例实际持有的契约内容不一致（契约的id也打出来了，这里是`"hello"`）。至于解决方案，这里给出了两种：
1. 增加微服务的版本，也就是要修改`microservice.yaml`配置文件里的`service_description.version`配置项的值
2. 将环境更改为开发环境，也就是在`microservice.yaml`配置文件里加上一个配置 `service_description.environment=development`

那么这里的两种问题解决方式应该选择哪一种呢，还有其他的解决方式吗？我们还是得先从服务契约本身说起。

一个 ServiceComb-Java-Chassis 框架的微服务在启动时会扫描它全部的REST接口类，并根据这些接口类生成对应的服务契约，并确保它们随着自己的服务记录一起注册到了服务中心。服务契约不仅仅是一份“接口文档”，同时它也约束着微服务的运行时行为。微服务会根据服务契约来确定如何调用其他服务，以及如何执行参数的序列化、反序列化操作等等。
也就是说， ServiceComb-Java-Chassis 的服务契约，不是那种大家开发完新功能后想起来要更新了就去改一下，忘记了或者没时间就不改的“接口说明文档”。它跟服务的接口有着严格的对应关系。**如果契约不一样的话，那么就说明，你的服务，真的，有接口变化**。

说到接口变化，大家都知道这是一个比较严肃的事情了。通常一个服务的接口都应该是经过设计和评审的，不能轻易改变（跟其他服务联调过的同学对这个应该有体会吧？）。
在正式的环境（生产环境）下，服务接口变更发生在什么时候呢？答案是，服务版本变更的时候。抛开那些不按套路出牌的开发团队不谈，一般情况下一个服务版本应该上线哪些功能，提供哪些接口，都是应该有规划和记录的。也就是说，一个服务的每个版本的接口都是确定的，不应该出现一个版本的服务有两套不同的接口的情况。

到这里，大家知道为什么你会碰到本文开头碰到的问题了吧？
原因很明确，就是**你不应该在一个版本的微服务里给出两套不相同的接口**。想想如果你的生产环境里，同一个版本的微服务混进去了一批接口不一样的服务实例（微服务架构下通常都是多实例部署服务的），那么你的系统一会儿可以正常工作，一会儿报各种参数错误、接口不存在之类的问题。这些问题的表象可能会显得很隐晦、随机或者复杂，而你在线上环境里很难排查出来。
为了避免将这类问题引入到线上， CSEJavaSDK / ServiceComb-Java-Chassis 采取了严格的服务契约校验策略，确保你的线上的服务一定是接口一致的。

所以异常里给出的两种解决方案其实也是针对不同的场景的：
1. 如果你正在开发新版本的服务，接口本来就变来变去的没定下来，那么你可以把你的微服务的环境改为开发环境（`service_description.environment=development`），此时微服务框架允许你变更你的接口并更新注册到服务中心。
2. 如果是测试、生产环境，这个时候服务接口应该稳定下来了，服务契约不同通常意味着这应该是一个新版本。那么你应该给你的服务分配一个更新的版本号了。

顺带一提，开发自测时还有一种问题解决办法，就是直接把服务中心里的旧服务记录删掉，更简单 ; )