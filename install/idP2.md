# idP 2 安装

根据 Shibboleth 官方通告，idP2 已经进入 End of life 周期。其将在 2016.7.31 停止更新。

idP3 版本的变化很大，在决定将老版本升级至 idP3 之前。针对 idP2 的文档依然有其价值。

#### 准备工作

- 一张 SSL 证书以开启 idP 的 https 访问
- 一个 metadata 的获取源，用于和互信的 sp 进行通讯。通常你可以向联盟的管理方去咨询这个，或者你也可以手动创建自己的 metadata。

安装时候它会自动为你生成以下信息：
- idP 的 entity ID
- 一对用于 SAML 通信时用的密钥 （不是用来 https 访问idP的）
- idP 的初始 metadata
- idP 的初始默认配置

#### 特别注意

- Ret hat/CentOS 一些老的版本里，默认使用的 GNU Java 编译的VM(gcj)。这是不支持的，请务必重新安装其他的JVM（比如 Oracle jdk）
- Debian7（“wheezy") 下的 使用 openjdk6 有一些问题，建议升级到 openjdk7
- 关于 openjdk，我们还是建议使用 Oracle 官方的标准 jdk。openjdk 有时候会有些意料之外的问题，比如内存泄漏，比如上面 Debian 的问题。你也可以猜到到对任何莫名其妙问题的解释都将指向在在Oracle的JVM上重新部署。

#### 安装 idP

Shibboleth idp V2 的版本是一个标准的 Java web application，基于 Servlet 2.4 规范。所以你可以在所有兼容此规范的 Servlet 容器上运行 idP 。如果你不知道该选哪一个，很多人使用 Apache Tomcat，也有很多人用 Jetty。

官方支持的容器版本是 Jetty7+ 和 Tomcat6+ 的版本。本文将以 Tomcat 为例，关于容器本身的一些文档，可以在对应的官网去查。

1. 下载 [idP](http://shibboleth.net/downloads/identity-provider/2.4.4/) 
2. 解压你下载的文件，例如 : 
	```
	unzip shibboleth-identityprovider-2.4.4-bin.zip
	```
3. 进入解压的目录，例如：
	```
	cd shibboleth-identityprovider-2.4.4
	```
4. 运行 ```./install.sh``` (linux) 或者 ```./install.bat``` (windows)
	- idP 的安装目录，本文中以 IDP_HOME 代替
5. 部署你的 idP war 文件到 java 容器里，文件的位置在 ```IDP_HOME/war/idp.war```

注意，idP 的 metadata 并不会在你修改了 idP 的配置后自动更新，你需要手动去更新这个

##### Tomcat 调优(可选）

环境部署
- 至少需要 Apache Tomcat 6.0.17 以上的版本
- Java 6 以上的版本

性能调优
- 添加以下参数到 JAVA_OPTS 的环境变量
	-  ```-Xmx512m``` 这里配置了你 Tomcat 所允许使用的最大内存，建议至少 512M
	-  ```-XX:MaxPermSize=128m``` JVM最大允许分配的非堆内存，按需分配
- 限制 Post 表单的大小。在 Tomact 的 Connector 中（AJP 的或者 http 的）配置 maxPostSize参数。100K(100000)是一个相对比较合理的数值。

部署 idp.war 时，最简单的办法当然是直接复制到 Tomcat 的 webapp 目录下，Tomcat 启动的时候会自动解压他。

另一种办法是告诉 Tomcat 到哪里去加载 idp.war。新建如下文件 ```TOMCAT_HOME/conf/Catalina/localhost/idp.xml``` 内容如下（替换IDP_HOME为你的实际路径)
```xml
<Context docBase="IDP_HOME/war/idp.war"
         privileged="true"
         antiResourceLocking="false"
         antiJARLocking="false"
         unpackWAR="false"
         swallowOutput="true" />
```

#### 使用反向代理

idP 部署在 tomcat 上，可以设置 http 对 tomcat 的代理，以便于维护和增强扩展性。
这里以 CentOS6 下 使用 apache 做 ajp 反向代理为例

取消 tomcat 的 AJP 身份认证
修改```/opt/apache-tomcat-6.0.36/conf/server.xml```
在 ```<!-- Define an AJP 1.3 Connector on port 8009 -->```下增加
```xml
request.tomcatAuthentication="false" address="127.0.0.1"
```

配置 apache 的 ajp 反向代理
修改 httpd.conf
```ProxyPass /idp/ ajp://localhost:8009/idp```

#### 部署 ssl 证书

在 apache 上部署 ssl 证书可比 tomcat 容易多了

查看```/etc/httpd/conf.d/ssl.conf```
在对应路径下部署证书

#### idP 状态的查看

可以访问这个地址 https://idp.example.edu.cn/idp/profile/Status
如果 idP 服务正常，那么应该显示 ok

也可以访问 https://idp.exmaple.edu.cn/idp/status
可以查看更详细的 idP 状态信息

后者默认只允许本机访问。可以修改 web.xml 来修改ACL。

#### 服务的启停

Tomcat服务的启停
```
/opt/apache-tomcat-6.0.36/bin/startup.sh
/opt/apache-tomcat-6.0.36/bin/shutdown.sh
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
/opt/apache-tomcat-6.0.36/bin/startup.sh
```