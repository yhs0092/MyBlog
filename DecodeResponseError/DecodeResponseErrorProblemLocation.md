> 本文基于CSEJavaSDK-2.3.35版本进行描述，对应的ServiceComb-Java-Chassis版本是1.1.0.B006。
> 文中的示例业务日志和代码来自[问题复现demo][问题复现demo]。

## 问题描述

问题复现demo在[这里][问题复现demo]。

前几天被拉去看一个问题。某服务（后面称其为A服务）采用同步模式运行，RPC方式调用其他微服务。在本地调试无问题，线上运行时此服务调用另外一个服务（后面称其为B服务）的接口会报错，且通过他们自定义扩展的一个`HttpClientFilter`的日志来看，被调用的provider服务已经正常返回了应答消息，但是在后面会报`ClassCastException`，无法将`InvocationException`转型为业务代码的返回值类型。日志如下：
```
// 业务逻辑被调用
[INFO] test() is called! com.github.yhs0092.blogdemo.javachassis.service.ConsumerService.test(ConsumerService.java:20)
// 用户自定义的HttpClientFilter中打印了provider返回的消息
[INFO] get response, status[200], content is [{"content":"returnOK"}] com.github.yhs0092.blogdemo.javachassis.filter.PrintResponseFilter.afterReceiveResponse(PrintResponseFilter.java:26)
// ClassCastException被抛出
[ERROR] invoke failed, invocation=PRODUCER rest client.consumer.test org.apache.servicecomb.swagger.invocation.exception.DefaultExceptionToResponseConverter.convert(DefaultExceptionToResponseConverter.java:35)
java.lang.ClassCastException: org.apache.servicecomb.swagger.invocation.exception.InvocationException cannot be cast to com.github.yhs0092.blogdemo.javachassis.service.TestResponse
	at com.sun.proxy.$Proxy30.test(Unknown Source)
	at com.github.yhs0092.blogdemo.javachassis.service.ConsumerService.test(ConsumerService.java:21)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.servicecomb.swagger.engine.SwaggerProducerOperation.doInvoke(SwaggerProducerOperation.java:160)
	at org.apache.servicecomb.swagger.engine.SwaggerProducerOperation.syncInvoke(SwaggerProducerOperation.java:148)
	at org.apache.servicecomb.swagger.engine.SwaggerProducerOperation.invoke(SwaggerProducerOperation.java:115)
	at org.apache.servicecomb.core.handler.impl.ProducerOperationHandler.handle(ProducerOperationHandler.java:40)
```
分析问题的过程中，他们提到由于线上的B服务还是旧版本的没有升级，于是他们把A服务依赖的B服务的接口jar包替换成了低版本来启动的。

## 分析过程

初步接触这个问题给人一种很怪异的感觉。如果一个consumer调用provider时都已经拿到了应答，那么会直接把应答返回给consumer的业务逻辑代码；万一中间真的出错了，那产生的`InvocationException`也应该是被“抛”出去的，而不是像日志里面显示的那样，尝试“返回”给consumer的业务逻辑才对。

可供分析的信息太少了，只能回头看一下sdk代码的相关逻辑，看看能不能复现出这个问题。

RPC调用模式的微服务里，业务逻辑通过provider接口做调用时，实际是通过ServiceComb生成的provider接口类型的代理来做调用的。而在这个代理的背后，实际调用流程的源头在`org.apache.servicecomb.provider.pojo.Invoker`类里面。同步调用模式下，区分应答如何被返回给业务逻辑的关键代码在`syncInvoke`方法里：
```java
protected Object syncInvoke(Invocation invocation, SwaggerConsumerOperation consumerOperation) {
  Response response = InvokerUtils.innerSyncInvoke(invocation);
  if (response.isSuccessed()) {
    // 在这里，response内的result会作为正常应答返回给业务逻辑
    return consumerOperation.getResponseMapper().mapResponse(response);
  }
  // 这里是异常逻辑，response内的result即为错误信息，会被包装为InvocationException抛给业务逻辑
  throw ExceptionFactory.convertConsumerException(response.getResult());
}
```
出现了线上日志中的错误说明这个方法没有走到throw语句，而是走return语句那里返回了。

`InvokerUtils.innerSyncInvoke()`方法里触发的主要流程是Handler->HttpClientFilter->网络线程，既然在用户自定义的`HTTPClientFilter`实现类的`afterReceiveResponse()`方法中已经打印出了B服务返回的应答消息，那么网络线程部分的嫌疑就可以排除了。问题只可能出在`Invoker`、`Handler`、`HTTPClientFilter`这三块。这个异常需要被catch住并塞到`response`里。同时，为了让异常作为response body返回，而不是被“抛”出去，`response.isSuccessed()`需要返回`true`，这就要求`response`的Http状态码必须是2xx的。通过在demo中加入自定义的`HttpClientFilter`，在`afterReceiveResponse()`方法中抛出一个状态码为200的`InvocationException`，我们复现出了这个问题，其日志特征与A服务的线上日志一致。

## 根因确定

一个response，里面装着一个异常，Http状态码却是2xx的，这个场景应该是不会发生的才对。在向A服务的开发同学确认了他们没有在自定义的`Handler`、`HttpClientFilter`内直接操作response后，我们通过日志也无法给出问题结论，只能等A服务的开发同学本地复现问题场景了。

好在这个问题本地是能够复现出来的，根因是在于A服务依赖的B服务接口jar包被替换后，旧版本的业务接口应答类型比新版本的多一个属性，而且这个属性的类型是找不到的，大致像下面这样：
```java
class ResponseType {
  private InnerFieldType someField; // 这里的InnerFieldType会报ClassNotFound
}
```
于是当`DefaultHttpClientFilter`的`extractResult()`方法尝试将Http body中的json串反序列化为业务代码中的应答对象时，会抛出一个异常，而这个异常被包装成`InvocationException`后，是被“return”回去的，而不是“throw”出去的，并且这个过程中没有打印任何日志。关键代码在`DefaultHttpClientFilter`的85-89行：
```java
try {
  return produceProcessor.decodeResponse(responseEx.getBodyBuffer(), responseMeta.getJavaType());
} catch (Exception e) {
  return ExceptionFactory.createConsumerException(e); // 异常被返回
}
```
“return”回去的异常被作为正常的应答对象塞进了`response`中，而`response`的状态码是Http应答的状态码——200，于是就有了线上碰到的错误。

## 总结

ServiceComb框架在此次定位过程中暴露出来的缺少日志的问题会在后续版本中修复。但是对于开发者而言，更重要的是服务上线部署前需要做好充分验证，临时替换依赖jar包这种简单粗暴的处理方式不可取。

那么本地调试过程中碰到这种问题应该如何定位呢？以本文所描述的场景（RPC调用方式，同步运行模式）来看，当业务代码中触发一次微服务调用，ServiceComb的处理流程大致是：
> Invoker -> InvokerUtils -> Handler -> HTTPClientFilter -> 网络线程

`Invoker`是RPC调用模式下的动态代理，业务代码通过provider接口做调用时，参数首先被传到`invoke()`方法中。由于consumer工作于同步模式，`Invoker`会通过`syncInvoke()`方法调用`InvokerUtils`的`innerSyncInvoke()`方法。在这里，`Invocation`的`next()`方法被调用，***从而触发Handler链执行***。在Handler链的末尾是`TransportClientHandler`，它会调用对应的transport方式发送请求。在Rest over Vertx传输方式下，我们需要关注的是`RestClientInvocation`的`invoke()`方法，这里会***遍历执行HttpClientFilter的beforeSendRequest()方法***，然后将请求调度到网络线程中发送。业务线程此时处于等待返回的状态（`SyncResponseExecutor.waitResponse()`方法中使用`CountDownLatch`进行等待）。

当请求应答返回后，`RestClientInvocation.processResponseBody()`方法会将Http response body返回给业务线程处理（通过触发`SyncResponseExecutor`的`CountDownLatch`）。应答首先会在`RestClientInvocation`中***遍历HttpClientFilter的afterReceiveResponse()方法***进行处理，然后***经过Handler链***的回调处理，最终返回给`InvokerUtils`的`syncInvoke()`方法。其中，***Http response body是在DefaultHttpClientFilter的extractResult()方法中反序列化为业务接口返回对象的***。这个方法会根据`response`的HTTP状态码判断如何对待结果，如果是2xx的状态码，则`response`中的`result`会作为正常的应答返回给业务逻辑，否则会将`result`包装到`InvocationException`中抛给业务逻辑。
```java
  // RestClientInvocation中处理应答的关键方法
  protected void processResponseBody(Buffer responseBuf) {
    invocation.getResponseExecutor().execute(() -> {
      // 同步模式下，应答返回流程从这里开始就是在业务线程里执行的
      try {
        HttpServletResponseEx responseEx =
            new VertxClientResponseToHttpServletResponse(clientResponse, responseBuf);
        for (HttpClientFilter filter : httpClientFilters) {
          // HttpClientFilter处理返回消息体，普通的filter会返回null
          Response response = filter.afterReceiveResponse(invocation, responseEx);
          if (response != null) { // DefaultHttpClientFilter会把消息体反序列化为应答对象，装入response返回
            asyncResp.complete(response); // 通过回调触发handler链
            return;
          }
        }
      } catch (Throwable e) {
        asyncResp.fail(invocation.getInvocationType(), e); // 包装异常，通过回调触发handler链
      }
    });
  }
```

本地分析这类问题的时候，首先需要知道请求发送的流程，了解RPC动态代理的入口、`Handler`链的起止点、`HttpClientFilter`的调用点。这些是流程中的关键节点，根据这些信息可以大致确定问题出现的范围。至于更进一步的定位，就需要大家根据具体的问题进行分析了。

> 看不懂上面说的是什么？正常 :P
>
> 只看一篇博客是很难弄懂这段流程的。关键的代码节点已经给出来了，自己写个demo调试一下，你的了解会更深刻
>
> (￣▽￣)ﾉ

[问题复现demo]: https://github.com/yhs0092/CSEBlogDemo-DecodeResponseError
