# 使用keytool和OpenSSL自行签发TLS证书

> 本文适用于Java微服务场景下，自行创建CA和签发TLS证书，方便自行开发调试，不适合于生产条件。  
> 最终目录结构：
>```
> workdir    // 这是本文的工作目录，可以随意自定义
> |- ca      // CA证书所在目录（根证书）
> |- cert    // 服务端/客户端身份证书所在目录
> |- trust   // 信任证书所在目录
>```

## 基本说明

创建一份证书，基本步骤有三步：

1. 创建一份自己的私钥文件

  理论上讲这份私钥文件，必须保密，不能让其他人获取到内容的。

2. 生成一份“证书请求文件”

  生成证书请求文件的后缀一般是`.csr`(Certificate Signing Request)。  
  如果你要购买一份生产环境使用的TLS证书，就需要签一份csr文件发给CA，让他们根据你的csr文件签发证书给你。

3. 签发证书

  由于本文创建的证书只是用于开发、测试目的，所以不需要向CA购买证书，而是自行创建CA证书，再用CA证书签发TLS证书。

在创建完TLS证书（身份证书）后，我们还可以使用JDK自带的`keytool`创建一份信任证书，这样可以让这套证书用于Java服务TLS对端认证的测试场景。

--------------------------------------------------------

环境信息：

- OpenJDK 1.8.0_191
- OpenSSL 1.1.0g

## 创建CA

跳转到 ca 目录下。由于是自己当CA签证书，所以要先创建好CA的证书。

### 创建CA证书私钥

命令`openssl genrsa -aes128 -out ca.key 4096`表示以aes128加密的方式生成一个长度为4096bit的RSA私钥文件。  
**注意**：创建过程中`openssl`会让你输入私钥文件的密码，进行加密。
```shell
workdir/ca# openssl genrsa -aes128 -out ca.key 4096
Generating RSA private key, 4096 bit long modulus
.............................................................................++
.....................................++
e is 65537 (0x010001)
Enter pass phrase for ca.key:
Verifying - Enter pass phrase for ca.key:

workdir/ca# ll
total 12
drwxr-xr-x 2 root root 4096 Nov 12 23:09 ./
drwxr-xr-x 3 root root 4096 Nov 12 23:08 ../
-rw------- 1 root root 3326 Nov 12 23:09 ca.key
```
执行完成后当前目录下应该生成了一份`ca.key`文件，这份文件是私钥，应该保密存储。

### 创建CA证书

命令`openssl req -new -x509 -key ca.key -out ca.crt -days 3650` 的含义为，使用CA私钥（`-key ca.key`），创建一份CA证书（`-out ca.crt`），有效时间为3650天（`-days 3650`）。  
**注意**：这一步需要输入CA私钥文件的密码，即上一步创建`ca.key`文件时设置的密码。
```shell
workdir/ca# openssl req -new -x509 -key ca.key -out ca.crt -days 3650
Enter pass phrase for ca.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Guangdong
Locality Name (eg, city) []:Shenzhen
Organization Name (eg, company) [Internet Widgits Pty Ltd]:xxx
Organizational Unit Name (eg, section) []:xxx
Common Name (e.g. server FQDN or YOUR name) []:xxx
Email Address []:xxx@xxx.xxx

workdir/ca# ll
total 16
drwxr-xr-x 2 root root 4096 Nov 12 23:21 ./
drwxr-xr-x 3 root root 4096 Nov 12 23:08 ../
-rw-r--r-- 1 root root 2069 Nov 12 23:21 ca.crt
-rw------- 1 root root 3326 Nov 12 23:09 ca.key
```
创建证书过程中需要输入所在地、公司名称，因为本文生成证书的目的只是开发测试，所以这里是随便填写的。执行完成后能够看到一份`ca.crt`文件。

## <span id="签发身份证书">签发身份证书</span>

跳转到 cert 目录下。

### 生成私钥

操作和刚才生成CA私钥一样，不再赘述。
```shell
workdir/cert# openssl genrsa -aes128 -out server.key 4096
Generating RSA private key, 4096 bit long modulus
.............................++
........................................................................................................................................................................................................................++
e is 65537 (0x010001)
Enter pass phrase for server.key:
Verifying - Enter pass phrase for server.key:

workdir/cert# ll
total 12
drwxr-xr-x 2 root root 4096 Nov 12 23:32 ./
drwxr-xr-x 4 root root 4096 Nov 12 23:31 ../
-rw------- 1 root root 3326 Nov 12 23:32 server.key
```

### 创建证书请求文件

创建证书请求文件需要用到上一步创建的私钥文件（`server.key`)。`A challenge password`和`An optional company name`那里可以直接敲回车跳过不填。
```shell
workdir/cert# openssl req -new -key server.key -out server.csr
Enter pass phrase for server.key:
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Guangdong
Locality Name (eg, city) []:Shenzhen
Organization Name (eg, company) [Internet Widgits Pty Ltd]:xxx
Organizational Unit Name (eg, section) []:xxx
Common Name (e.g. server FQDN or YOUR name) []:xxx
Email Address []:xxx@xxx.xxx

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:

workdir/cert# ll
total 16
drwxr-xr-x 2 root root 4096 Nov 12 23:42 ./
drwxr-xr-x 4 root root 4096 Nov 12 23:31 ../
-rw-r--r-- 1 root root 1724 Nov 12 23:42 server.csr
-rw------- 1 root root 3326 Nov 12 23:32 server.key
```
执行完成后目录下会出现一份`server.csr`文件。

### 签发证书文件

注意这里执行的命令会指定使用上面创建的CA证书以及CA私钥文件（`-CA ../ca/ca.crt -CAkey ../ca/ca.key`），有效期为3650天，输出文件为`server.crt`。
```shell
workdir/cert# openssl x509 -req -days 3650 -in server.csr -CA ../ca/ca.crt -CAkey ../ca/ca.key -CAcreateserial -out server.crt
Signature ok
subject=C = CN, ST = Guangdong, L = Shenzhen, O = xxx, OU = xxx, CN = xxx, emailAddress = xxx@xxx.xxx
Getting CA Private Key
Enter pass phrase for ../ca/ca.key:

workdir/cert# ll
total 24
drwxr-xr-x 2 root root 4096 Nov 12 23:49 ./
drwxr-xr-x 4 root root 4096 Nov 12 23:31 ../
-rw-r--r-- 1 root root 1948 Nov 12 23:49 server.crt
-rw-r--r-- 1 root root 1724 Nov 12 23:42 server.csr
-rw------- 1 root root 3326 Nov 12 23:32 server.key
-rw-r--r-- 1 root root   17 Nov 12 23:49 .srl
```

### 将crt证书转换为PKCS12格式

上一步其实已经成功创建出身份证书了（就是`server.crt`），这里把该文件转换成PKCS12格式，方便Java服务使用。  
**注意**：这一步需要输入私钥`server.key`的密码，导出证书的密码可由读者自行设置。
```shell
workdir/cert# openssl pkcs12 -export -in server.crt -inkey server.key -out server.p12
Enter pass phrase for server.key:
Enter Export Password:
Verifying - Enter Export Password:

workdir/cert# ll
total 32
drwxr-xr-x 2 root root 4096 Nov 13 00:09 ./
drwxr-xr-x 5 root root 4096 Nov 12 23:52 ../
-rw-r--r-- 1 root root 1948 Nov 12 23:49 server.crt
-rw-r--r-- 1 root root 1724 Nov 12 23:42 server.csr
-rw------- 1 root root 3326 Nov 12 23:32 server.key
-rw------- 1 root root 4157 Nov 13 00:09 server.p12
-rw-r--r-- 1 root root   17 Nov 12 23:49 .srl
```

## 创建信任证书

跳转到trust目录。

### 创建keystore文件

使用keytool新建一份 keystore 文件，这个文件会被用作信任证书：
```shell
workdir/trust# keytool -genkeypair -alias trust -keystore trust.jks -keyalg RSA -sigalg SHA1withRSA
Enter keystore password:
Re-enter new password:
What is your first and last name?
  [Unknown]:  xxx
What is the name of your organizational unit?
  [Unknown]:  xxx
What is the name of your organization?
  [Unknown]:  xxx
What is the name of your City or Locality?
  [Unknown]:  Shenzhen
What is the name of your State or Province?
  [Unknown]:  Guangdong
What is the two-letter country code for this unit?
  [Unknown]:  CN
Is CN=xxx, OU=xxx, O=xxx, L=Shenzhen, ST=Guangdong, C=CN correct?
  [no]:  yes
```
**注意**：创建过程中`keytool`会要求你给这份 keystore 文件设置密码。
执行成功的话会在trust目录下生成一份 keystore 文件：
```
workdir/trust# ll
total 12
drwxr-xr-x 2 root root 4096 Nov 12 23:54 ./
drwxr-xr-x 5 root root 4096 Nov 12 23:52 ../
-rw-r--r-- 1 root root 2557 Nov 12 23:54 trust.jks
```

### 导入CA证书

将前文创建的CA证书导入到`trust.jks`文件中，这样，使用此CA证书签发的TLS证书就都被这份 keystore 文件信任了。  
**注意**：导入过程需要输入`trust.jks`文件的密码，也可以在命令中使用`-storepass`参数输入密码。
```shell
workdir/trust# keytool -import -file ../ca/ca.crt -keystore trust.jks
Enter keystore password:
Owner: EMAILADDRESS=xxx@xxx.xxx, CN=xxx, OU=xxx, O=xxx, L=Shenzhen, ST=Guangdong, C=CN
Issuer: EMAILADDRESS=xxx@xxx.xxx, CN=xxx, OU=xxx, O=xxx, L=Shenzhen, ST=Guangdong, C=CN
Serial number: d66ebd9ea172d18a
Valid from: Tue Nov 12 23:21:27 CST 2019 until: Fri Nov 09 23:21:27 CST 2029
Certificate fingerprints:
         SHA1: A2:19:67:E6:D2:A2:6E:7E:E6:C8:15:2C:CD:F7:48:70:EF:85:03:8D
         SHA256: 58:FF:94:50:E4:22:73:A3:8C:77:7D:6E:06:38:D6:98:0D:0D:58:83:44:46:93:CB:4E:56:48:AE:AC:41:63:F0
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 4096-bit RSA key
Version: 3

Extensions:

#1: ObjectId: 2.5.29.35 Criticality=false
AuthorityKeyIdentifier [
KeyIdentifier [
0000: 95 4E 27 2B 65 7C E6 8F   E3 F8 46 7D 09 3B 44 ED  .N'+e.....F..;D.
0010: 08 60 94 00                                        .`..
]
]

#2: ObjectId: 2.5.29.19 Criticality=true
BasicConstraints:[
  CA:true
  PathLen:2147483647
]

#3: ObjectId: 2.5.29.14 Criticality=false
SubjectKeyIdentifier [
KeyIdentifier [
0000: 95 4E 27 2B 65 7C E6 8F   E3 F8 46 7D 09 3B 44 ED  .N'+e.....F..;D.
0010: 08 60 94 00                                        .`..
]
]

Trust this certificate? [no]:  yes
Certificate was added to keystore
```
执行成功后，可以运行`keytool -list -v -keystore trust.jks`命令查看导入的内容。

## 完成

至此，证书生成工作就完成了。对于Java服务，可以使用`cert`目录下的`server.p12`作为身份证书，`trust`目录下的`trust.jks`作为信任证书来启动服务。
`trust.jks`信任了签发`server.p12`的CA证书，所以客户端、服务端程序开启TLS对端认证进行测试时，可以使用同一套证书，
也可以重复[签发身份证书](#签发身份证书)的步骤签发多套证书。

## 附录

### 如何查看证书

- 使用keytool查看crt证书：
  
  `keytool -printcert -v -file server.crt`

- 使用keytool查看keystore文件：
  
  `keytool -list -v -keystore trust.jks`

- 使用OpenSSL查看crt证书：
  
  `openssl x509 -in server.crt -text -noout`

## 参考文档

- https://www.cnblogs.com/yjmyzz/p/openssl-tutorial.html
- https://sites.google.com/site/ddmwsst/create-your-own-certificate-and-ca
- https://blog.csdn.net/defonds/article/details/85098684
- https://blog.csdn.net/linvo/article/details/9150607