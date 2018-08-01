## 前言
用户开发的微服务要想注册到CSE的服务中心，就需要用到AK/SK认证。由于CSEJavaSDK提供了较多的配置方式，有时候容易出现错配和漏配的情况，本文从CSEJavaSDK读取AK/SK的关键代码入手进行分析，希望能够给大家提供一点AK/SK认证失败时的定位思路。（本文基于CSEJavaSDK-2.3.30进行说明）

## 代码逻辑分析

首先需要说明的是，AK/SK也是一个配置项，因此它可以配置在microservice.yaml文件里，可以通过-D设置系统属性来指定，也可以通过环境变量来指定（Windows的环境变量貌似可以带点号`.`，因此你可以直接在环境变量中指定cse.credentials.accessKey=ak；而Linux的环境变量不能带点，所以不能这么做）。另一方面来说，AK/SK又是一个比较特殊的配置项，因此CSEJavaSDK又提供了一个[加密存储](https://support.huaweicloud.com/devg-cse/cse_03_0088.html)的配置方式。无论AK/SK的来源是哪里，最终CSEJavaSDK都是在AKSKManager中完成AK/SK的读取逻辑的。
```java
public static Credentials getCredential() throws Exception {
  // 读取AK/SK
  AKSKOption option = AKSKOption.readAKSK();
  // 中间读写缓存等等逻辑忽略……
  // 根据cse.credentials.akskCustomCipher配置获取cipher，对AK/SK进行解密
    AKSKCipher cipher = (AKSKCipher)CIPHERS.get(option.getAkskCustomCipher());
  // 检查逻辑忽略...
    char[] ak = cipher.decode(TYPE.AK, option.getAccessKey().toCharArray());
    char[] sk = cipher.decode(TYPE.SK, option.getSecretKey().toCharArray());
    String project = option.getProject();
    if (project == null || project.isEmpty()) {
      // 如果用户没有配置cse.credentials.project，就尝试从配置的服务中心地址进行解析，具体的代码忽略……
      // 我们平常通过APIGateway连接到服务中心，例如你要连到华北区，配置的地址就是这样的 https://cse.cn-north-1.myhuaweicloud.com
      // 可以从中截取出 cn-north-1，这个就是华北区的project
      LOGGER.info("The application missing project infomation, so choose a nearest one [{}]", project);
    }
  return credentials;
}
```
以上是AKSKManager的getCredential方法逻辑，可以看到大体的流程是读取AK/SK、根据cse.credentials.akskCustomCipher配置项获取cipher对aksk进行解密、获取cse.credentials.project配置信息。此方法返回的credentials内包含了明文的AK/SK、project信息，因此***本地调试时确定AK/SK读取是否有问题的最快途径就是在getCredential方法末尾打断点，查看credentials包含的信息***。

而在AKSKOption.readAKSK()中读取AK/SK的关键流程如下：
```java
public static AKSKOption readAKSK() {
  // 从AK/SK加密存储文件读取
  AKSKOption option = readFromFile();
  if (option == null) {
    // 从配置中读取，注意：环境变量、System Property、配置文件都算在这里面
    option = buildFromYaml();
  }
  return option;
}
private static AKSKOption readFromFile() {
  // 忽略缓存逻辑……
  // 先尝试从环境变量CIPHER_ROOT中读取加密存储文件的目录
    String cipherPath = System.getenv("CIPHER_ROOT");
    if (cipherPath == null || cipherPath.isEmpty()) {
      // 若不存在则从/opt/CSE/etc/cipher目录中读取
      cipherPath = "/opt/CSE/etc/cipher";
    }
    String certFilePath = cipherPath + File.separator + "certificate.yaml";
    File certFile = new File(certFilePath);
    if (!certFile.exists() || certFile.isDirectory()) {
      // 文件不存在时尝试从系统属性user.dir配置的目录中读取
      certFile = new File(System.getProperty("user.dir") + File.separator + "certificate.yaml");
      if (!certFile.exists() || certFile.isDirectory()) {
        return null;
      }
    }
    AKSKOption option = readFromFile(certFile);
    return option;
}
```
可以看到CSEJavaSDK优先读取加密存储的AK/SK文件，读不到才去配置中找。而AK/SK加密文件的读取路径优先级从高到低分别是CIPHER_ROOT环境变量配置的目录、/opt/CSE/etc/cipher目录、user.dir系统属性配置的目录（这个少见）。

## 常见问题分析思路

当发生AK/SK认证失败的问题时，我们首先需要对问题有个基本的定界，即问题是出在AK/SK读取（解密）上，还是出在AK/SK的内容本身。

如果是本地开发调试，那么这个问题很好确定，直接在`AKSKManager`的`getCredential`方法里打断点去看一下即可。如果是在线上部署运行的服务，那么只能看日志，凭经验来定位了。与AK/SK认证相关的日志关键词有这些：
- "read ak/sk from"：显示AK/SK的来源是哪里，"read ak/sk from security storage file."表示服务实例从AK/SK加密存储文件中读取配置项的，"read ak/sk from microservice.yaml."表示服务实例从配置项中读物AK/SK的（注意，这里说的是***配置项，包含了环境变量、系统属性以及配置文件***，不单单是指microservice.yaml文件）。
- "add ak/sk cipher"：加载`AKSKCipher`，cipher用于解密SK。CSEJavaSDK默认提供的cipher有default、security，default用于明文存储AK/SK的场景，security在[AK/SK加密存储](https://support.huaweicloud.com/devg-cse/cse_03_0088.html)场景中使用。CSEJavaSDK使用SPI机制加载cipher，如果发现日志中没有打印加载default和security的内容，那么你就需要检查一下自己依赖的jar包了。
- "sign request failed ...... OSS_ROOT not properly set"：当cipher为`security`时，CSEJavaSDK会去CIPHER_ROOT环境变量配置的目录里读取解密密钥文件common_shared.key和root.key，如果用户忘记配置CIPHER_ROOT了，就会打印这个错误。
- "Cipher decode error, will use original as plain!"：SK解密失败时会打印此错误日志，常见的错误原因包括CIPHER_ROOT环境变量下没有密钥文件、程序没有权限读取密钥文件、Java没有开启高强度加密功能等。关于Java高强度加密功能，大家去网上搜一下JCE policy能找到解除限制的方法，较低版本的jdk8需要下载一个jce的jar包，高版本的直接修改jre目录下的lib/security/java.security文件配置即可，这个限制只存在于Oracle JDK中，OpenJDK没有此限制。
- AK/SK或project配置内容错误：当内容配置出错时，实际上AK/SK的读取加密过程并不会报错，只有等到CSEJavaSDK向CSE后端服务（sc/cc/monitor）发送请求时，才会拿到错误返回消息，错误信息包括如下类型：
```
AK内容错误：
401:Unauthorized, {"errorCode":"401002","errorMessage":"Request unauthorized","detail":"Get service token from iam proxy failed,{\"error\":\"get SK from AK from iam failed. error:Object Not Found - details: \"}"}
```
```
SK内容错误：
401:Unauthorized, {"errorCode":"401002","errorMessage":"Request unauthorized","detail":"Get service token from iam proxy failed,{\"error\":\"validate ak sk error\"}"}
```
```
project配置错误：
401:Unauthorized, {"errorCode":"401002","errorMessage":"Request unauthorized","detail":"Get service token from iam proxy failed,{\"error\":\"get project token from iam failed. error:http post failed, statuscode: 400\"}"}
```
这些错误内容都是华为云服务的认证鉴权机制返回的，后期如果相关服务升级的话，也许返回的错误内容会有变化。但总之大家可以抓住关键的一点，就是如果前面的AK/SK加载过程都没有报错，只有连接CSE后端服务时拿到401的错误返回，那么你就需要检查一下自己的AK/SK、project是否配置对了。-D启动参数、环境变量、各个microservice.yaml文件都是需要排查的。至于project的配置，可以参考[如何查看项目ID](https://support.huaweicloud.com/usermanual-iam/zh-cn_topic_0079477316.html)。project和你所连接的region是相关的，比如华北区连接的APIGateway地址就是https://cse.cn-north-1.myhuaweicloud.com ，project应该配置为cn-north-1。如果你不配置，CSEJavaSDK会尝试从sc地址中截取出来，但不一定能拿到正确的值，此时启动日志里面会打印提示信息，例如"The application missing project infomation, so choose a nearest one [cn-north-1]"，就是在告诉你自动选取的project是cn-north-1。

## 总结

博主对AK/SK认证的了解大致也就是上面这些了，很多内容也只能提供一点参考和启发意义。项目实际运行的情况复杂多样，当大家遇到相关问题的时候还是需要根据上面提到的代码逻辑，结合自己服务的实际运行环境去分析，才能更好地定位问题。
