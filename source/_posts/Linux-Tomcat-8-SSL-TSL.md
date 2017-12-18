---
title: Linux Tomcat 8 SSL/TSL(https证书)配置
date: 2017-10-21 20:58:21
categories: 笔记
tags: [Linux,Tomcat,SSL,TSL,HTTPS,签名,证书,秘钥]
---
## <center>Tomcat 8 SSL/TSL(https证书)配置<center>

最近公司要求在测试服务器上配置https访问方式。由于测试服务器，先使用java自签名证书测试一下，之后改成阿里云申请的免费证书，选择jks格式证书，修改keystoreFile、keystorePass即可。配置方法主要[参考Tomcat官方文档](https://tomcat.apache.org/tomcat-8.0-doc/ssl-howto.html#General_Tips_on_Running_SSL)。

<!--more-->

按照官方文档步骤：

* 准备证书秘钥
* 编辑 Tomcat 配置文件
* 疑难排解

## 创建一个keystore文件

先创建一个keystore文件保存服务器的私有密钥和自签名证书。

Windows：

`"%JAVA_HOME%\bin\keytool" -genkey -alias tomcat -keyalg RSA -keystore tomcat.keystore`

UNIX：

`$JAVA_HOME/bin/keytool -genkey -alias tomcat -keyalg RSA -keystore tomcat.keystore`

* 设置密码为“changeit”

windows命令将在user目录下创建一个.keystore后缀的文件。要想指定一个不同的位置或文件名，可以在上述的 keytool 命令上添加 -keystore 参数，后跟到达 keystore 文件的完整路径名。

linux下，测试服务器上配置的环境，所以使用keytool -genkey -alias tomcat -keyalg RSA -keystore tomcat.keystore,在当前目录下生产自签名文件

## 编辑配置 Tomcat

Tomcat 能够使用两种 SSL 实现：

* JSSE 实现，它是Java 运行时（从 1.4 版本起）的一部分。
* APR 实现，默认使用 OpenSSL 引擎。

这里我们选用JSSE实现。接下来我们开始修改配置文件

* 进入Tomcata主目录

        cd /home/test/apache-tomcat-8.0.44

* 编辑配置文件server.xml

        vi conf/server.xml

由于8.0.44是Tomcat 8的最新版本，配置文件发生了一些改变，老版本只要打开注释，新版本似乎不建议这样配置了，但是没关系，我们依然可以选择在

```
<!-- Define a SSL/TLS HTTP/1.1 Connector on port 8443
                This connector uses the NIO implementation that requires the JSSE
                style configuration. When using the APR/native implementation, the
                OpenSSL style configuration is required as described in the APR/native
                documentation -->
```

下面添加
```
        <Connector
                protocol="org.apache.coyote.http11.Http11NioProtocol"
                port="8443" maxThreads="200"
                scheme="https" secure="true" SSLEnabled="true"
                keystoreFile="${user.home}/.keystore" keystorePass="changeit"
                clientAuth="false" sslProtocol="TLS"/>
```

keystoreFile为签名文件位置，keystorePass为签名密码。
每个属性所用的配置信息与选项都是强制的，可查看 HTTP 连接器配置参考文档中的 SSL 支持部分。一定要确保对所使用的连接器采用正确的属性。BIO、NIO 以及 NIO2 连接器都使用 JSSE，可以在protocol属性选择使用哪一个，而APR以及原生的连接器则使用 APR。
BIO、NIO 以及 NIO2 可参考一下文档：
<http://blog.sina.com.cn/s/blog_aed82f6f010194ky.html>
<https://www.ibm.com/developerworks/cn/java/j-nio2-1>

port属性指的是 Tomcat 用以侦听安全连接的 TCP/IP 端口号。你可以随意改变它，比如改成 https 的默认端口号 443。

测试服务器占用端口比较多，我选用了10080和10443作为http和https的端口。

## 疑难排解

按照官方文档，按理来说应该可以正常启动运行。但是实际情况，使用https访问异常。DOC文档说，sslProtocol和sslEnabledProtocols有重叠，但是两个仍然必须设置，否则会有异常。

org.apache.tomcat.util.net.Nio2Endpoint.setSocketOptions 

javax.net.ssl.SSLHandshakeException: No appropriate protocol

[查阅文档](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html#SSL_Support)

选择支持TLSv1,TLSv1.1,TLSv1.2 [Java 7开始支持TLSv1.2]
添加参数sslEnabledProtocols和ciphers。选择TLS RSA加密方式。
！注意java 8的改变，具体参考上方文档。

---
## 最终配置
```
        <Connector port="10443" protocol="org.apache.coyote.http11.Http11Nio2Protocol"
               maxThreads="200" SSLEnabled="true" scheme="https"
               secure="true" clientAuth="false" sslProtocol="TLS"
               sslEnabledProtocols="TLSv1,TLSv1.1,TLSv1.2"
               keystoreFile="tomcat.keystore" keystorePass="changeit"
               ciphers="TLS_RSA_WITH_AES_128_CBC_SHA,
                        TLS_RSA_WITH_AES_256_CBC_SHA,
                        TLS_RSA_EXPORT1024_WITH_RC4_56_MD5,
                        TLS_RSA_WITH_AES_128_CBC_SHA256,
                        TLS_RSA_WITH_AES_256_CBC_SHA256,
                        TLS_RSA_WITH_RC4_128_MD5,
                        TLS_RSA_WITH_RC4_128_SHA,
                        TLS_RSA_WITH_DES_CBC_SHA,
                        TLS_RSA_WITH_3DES_EDE_CBC_SHA,
                        TLS_RSA_EXPORT1024_WITH_DES_CBC_SHA"/>
```