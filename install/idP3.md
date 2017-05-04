# idP 3 安装

## 环境需求

- Oracle Jave 或者 OpenJDK 7 或 8，需下载对应的 the Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files 文件。虽然 JRE 可能可以使用，但是技术团队只支持JDK。
- 需要支持Servlet API 3.0 的Servlet容器
	- Tomcat7+
	- Jetty8+
- 正式支持的版本 Tomcat 8.x和Jetty 9.2.x，尽管上文有提及 Tomcat7 等老版本，但并没有对以上版本进行测试，可能有 Bug。
- 由于核心技术团队是以Jetty平台进行开发测试的，因此官方推荐使用Jetty 9 容器，笔者个人依然倾向于使用更为熟悉的 Tomcat8。
- 无操作系统限制，但推荐使用Linux, OS X 和 Windows。

#### 特别注意

- Ret hat/CentOS 一些老的版本里，默认使用的 GNU Java 编译的VM(gcj)。这是不支持的，请务必重新安装其他的JVM（比如 Oracle jdk）
- 关于 openjdk，我们还是建议使用 Oracle 官方的标准 jdk。openjdk 有时候会有些意料之外的问题，比如内存泄漏，比如上面 Debian 的问题。你也可以猜到到对任何莫名其妙问题的解释都将指向在在Oracle的JVM上重新部署。

#### 不支持的版本
idP V3.0不支持以下Idp的老版本配置：

- Java6或更早的版本.
- 不支持Java Cryptography Extension (JCE) Unlimited Strength Jurisdiction Policy Files.
- Tomcat6或更早版本。请注意如系统已安装了RHEL5或RHEL6（RHEL5预装的是Tomcat5，RHEL6预装的是Tomcat6），在安装Idp 3.0时，则需要先安装RHEL7的一个可选应用容器，这样才能可以使用Tomcat7（因此我们推荐直接使用Tomcat8）.
- Jetty7或更早的版本.

## 安装

#### 准备工作
- 一张 SSL 证书以开启 idP 的 https 访问
- 用于表示你 idP 的 entityID，安装程序默认会使用你的 hostname）
- 一个 metadata 的获取源，用于和互信的 sp 进行通讯。通常你可以向联盟的管理方去咨询这个，或者你也可以手动创建自己的 metadata。

如果没有 metadata 的话，你可以在 [Testshib](http://www.testshib.org/) 上测试你的 idP。

安装程序会为你建议或生成一些信息：
- idP 的 entityID
- 一对自签发的密钥和证书，用于：
	- 消息传输的验证
	- 默认的 https 通讯加密，使用8443端口
	- 其他系统和 idP 的信息加密和解密
- idP 自己加密 cookies 和其他数据的 密钥和密钥版本文件。（这是 java keystore 的格式 "JCEKS")
- 基于这些信息产生的 idP 的默认配置

安装好 tomcat8 并做好 java 环境
```
cat /etc/profile
JAVA_HOME=/usr/java/jdk1.8.0_131
TOMCAT_HOME=/opt/apache-tomcat-8.0.43
PATH=$JAVA_HOME/bin:$PATH
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME
export TOMCAT_HOME
export PATH
export CLASSPATH
```
#### linux 下安装

Shibboleth idp 是一个标准的 Java web application，基于 Servlet 3.0 规范。你可以在所有兼容此规范的 Servlet 容器上运行 idP 。官方支持的版本是 Tomcat 和 Jetty，这里使用 Tomcat 为例。

1. 下载 [idP](http://shibboleth.net/downloads/identity-provider/latest) 最新的版本
2. 解压你下载的文件，例如 : 
	```
	unzip shibboleth-identityprovider-VERSION-bin.zip
	```
3. 进入解压的目录，例如：
	```
	cd shibboleth-identityprovider-VERSION
	```
4. 运行 ```./install.sh``` (linux) 或者 ```./install.bat``` (windows)
	- idP 的安装目录，本文中以 idp.home代替
5. 部署 idP 的 warfile。路径在  ```idp.home/war/idp.war```。后文会专门说明如何部署

##### Rebuild idP warfile
如果你要重新编译 idp warfie，运行 ```bin/build.sh``` 
```bash
opt/shibboleth-idp# bin/build.sh
```

#### Apache Tomcat 配置

##### 基本说明
本文中，idp.home 指代 idP 的安装路径，TOMCAT_HOME 指代 Tomcat 的安装路径

所有 Tomcat8 的版本都可以用，当然最好还是用[最新的版本](http://tomcat.apache.org/download-80.cgi)

##### 必须调整的配置

在 idP3 中，你必须指定 Tomcat 去读取 idP 的 warfile。创建 ```TOMCAT_HOME/conf/Catalina/localhost/idp.xml```文件，复制以下内容。（注意替换 idp.home 为你自己的 idP 安装路径）
```xml
<Context docBase="idp.home/war/idp.war"
         privileged="true"
         antiResourceLocking="false"
         swallowOutput="true">
 
    <!-- Work around lack of Max-Age support in IE/Edge -->
    <CookieProcessor alwaysAddExpires="true" />
 
</Context>
```
- Tomcat 默认监听 8080 和 8443 端口。一般我们需要把他改成 80 和 443，你可以在```TOMCAT_HOME/conf/server.xml```修改这个。本文示例中，我们将用 apache 来代理 Tomcat，所以不去管它。
- - Tomcat 默认没有提供 Java Server Tag Library，这使得 idP3 的 status 页面无法显示。解决的办法是下载 [jstl的jar包](https://build.shibboleth.net/nexus/service/local/repositories/thirdparty/content/javax/servlet/jstl/1.2/jstl-1.2.jar)，然后放在 idp.home/edit-webapp/WEB-INF/lib/ 内，然后需要重新 Build 一下 idp。在 idp.home 的目录下，```./bin/build.sh``` 即可
- 添加以下参数到 CATALINA_OPTS 的环境变量。
	- ```-Didp.home=<location> ``` <location> 替换为你实际的路径
	- v3.12以后的版本中，idp.home 也可以通过 web.xml 来指定。在 edit-webapp 目录下创建web.xml，然后重新 build idp.war（未测试）
	```xml		
	<context-param>
	    <param-name>idp.home</param-name>
	    <param-value>/opt/idp</param-value>
	</context-param>
	```
	-  ```-XX:+UseG1GC``` 开启垃圾回收器，以在 metadata 比较大的时候获得好一点的性能
	-  ```-Xmx1500m``` JVM 的最大内存，如果联盟的 metadata 很大（超过 25M) 那么至少需要 1.5G 的内存。最好根据实际情况测试一下
	-  ```-XX:MaxPermSize=128m``` JVM最大允许分配的非堆内存，按需分配

```
CATALINA_OPTS="-server -Didp.home=/opt/idp -XX:+UseG1GC -Xmx1500m -XX:MaxPermSize=128m"
export CATALINA_OPTS
```


##### 建议的配置调整
- 限制 Post 表单的大小。在 Tomact 的 Connector 中（AJP 的或者 http 的）配置 maxPostSize参数。100K(100000)是一个相对比较合理的数值。
- 关掉 Tomcat 的会话保持。在 ```TOMCAT_HOME/conf/context.xml```中取消 ``` <Manager pathname="" /> ``` 的注释。这可以防止在容器关闭时，idP 的会话错误。没必要通过 Tomcat 来为 idP 集群做会话保持。 

#### 使用反向代理

idP 部署在 tomcat 上，可以设置 http 对 tomcat 的代理，以便于维护和增强扩展性。
这里以 CentOS6 下 使用 apache 做 ajp 反向代理为例

取消 tomcat 的 AJP 身份认证（如果需要的话）
修改```TOMCAT_HOME/conf/server.xml```
在 ```<!-- Define an AJP 1.3 Connector on port 8009 -->```下增加
```xml
request.tomcatAuthentication="false" address="127.0.0.1"
```

配置 apache 的 ajp 反向代理
修改 httpd.conf
```ProxyPass /idp/ ajp://localhost:8009/idp/```

#### 部署 ssl 证书

在 apache 上部署 ssl 证书可比 tomcat 容易多了

查看```/etc/httpd/conf.d/ssl.conf```
在对应路径下部署证书

#### idP 状态的查看

```idp.home/bin/status.sh```
```
### Operating Environment Information
operating_system: Linux
operating_system_version: 2.6.32-696.el6.x86_64
operating_system_architecture: amd64
jdk_version: 1.8.0_131
available_cores: 2
used_memory: 749 MB
maximum_memory: 1500 MB

### Identity Provider Information
idp_version: 3.3.1
start_time: 2017-05-04T00:23:39+08:00
current_time: 2017-05-04T00:24:01+08:00
uptime: 21304 ms

service: shibboleth.LoggingService
last successful reload attempt: 2017-05-03T16:22:09Z
last reload attempt: 2017-05-03T16:22:09Z

service: shibboleth.ReloadableAccessControlService
last successful reload attempt: 2017-05-03T16:22:16Z
last reload attempt: 2017-05-03T16:22:16Z

service: shibboleth.MetadataResolverService
last successful reload attempt: 2017-05-03T16:22:16Z
last reload attempt: 2017-05-03T16:22:16Z

service: shibboleth.RelyingPartyResolverService
last successful reload attempt: 2017-05-03T16:22:14Z
last reload attempt: 2017-05-03T16:22:14Z

service: shibboleth.NameIdentifierGenerationService
last successful reload attempt: 2017-05-03T16:22:14Z
last reload attempt: 2017-05-03T16:22:14Z

service: shibboleth.AttributeResolverService
last successful reload attempt: 2017-05-03T16:22:14Z
last reload attempt: 2017-05-03T16:22:14Z

        DataConnector staticAttributes: has never failed

service: shibboleth.AttributeFilterService
last successful reload attempt: 2017-05-03T16:22:14Z
last reload attempt: 2017-05-03T16:22:14Z
```

#### 时间同步

idP 服务队时间敏感，因此务必配置 ntp 服务同步时间。上海教育城域网用户可使用 ntp.shec.edu.cn

#### 服务的启停

Tomcat服务的启停
```
TOMCAT_HOME/bin/startup.sh
TOMCAT_HOME/bin/shutdown.sh
```
Apache服务的启停
```
Service start httpd
Service stop httpd
```

可以设置为系统自动启动服务，仍以 CentOS 6 为例
```
Chkconfig httpd on
Chkconfig ntpd on
```
修改```/etc/rc.local```
添加
```
TOMACT_HOME/bin/startup.sh
```

