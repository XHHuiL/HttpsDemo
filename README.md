### Spring Boot配置SSL实现https请求

#### 1. 生成SSL证书
+ 专业的SSL证书较为昂贵，可以在腾讯云或者阿里云上申请免费的SSL证书。
+ 如果只是做简单的demo，可以使用java自带的keytool工具生成SSL证书。

#### 2. 使用keytool工具生成SSL证书
以windows系统为例（如果是linux系统，将keytool.exe替换为keytool即可），在终端输入
```text
keytool.exe -genkey -alias test -storetype PKCS12 -keyalg RSA -keysize 2048 -keystore test.p12 -validity 3650
```

参数解释
```text
-genkey: 生成SSL证书
-alias: 证书别名
-storetype: 秘钥仓库类型
-keyalg: 生成证书算法
-keysize: 证书大小
-keystore: 生成证书保存路径
-validity: 证书有效期
```

注意在终端输入生成SSL命令后，“您的名字和姓氏是什么？”这一项需要设置为你的域名，本地测试的话便是localhost。

#### 3. Spring Boot项目配置SSL
在配置文件application.yml中添加如下内容
```text
server:
#  配置端口号，https默认端口号为443，如果443端口被占用，将占用443端口的进程杀死
  port: 443
#  配置ssl证书
  ssl:
#    SSL证书test.p12与application.yml放在同级目录下
    key-store: classpath:test.p12
    key-store-password: 123456
    keyStoreType: PKCS12
    keyAlias: test
```

#### 4. 测试SSL是否配置成功
创建HelloController
``` java
@RestController
@CrossOrigin(origins = "*")
public class HelloController {

    @GetMapping(value = "/hello")
    public String sayHello() {
        return "Hello World!";
    }

}
```

使用浏览器访问https://localhost/hello ，看是否正确响应
（第一次访问时，浏览器会警告，这是因为自己生成的SSL证书不被浏览器认可）。

如果出错：检查application.yml中端口号是否为443，检查443端口是否被其他进程占用。

#### 5. http请求自动转换为https请求
修改入口类HttpsApplication
``` java
package com.example.https;

import org.apache.catalina.Context;
import org.apache.catalina.connector.Connector;
import org.apache.tomcat.util.descriptor.web.SecurityCollection;
import org.apache.tomcat.util.descriptor.web.SecurityConstraint;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.context.annotation.Bean;


@SpringBootApplication
public class HttpsApplication {

    public static void main(String[] args) {
        SpringApplication.run(HttpsApplication.class, args);
    }

    @Bean
    public Connector connector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        // 捕获http请求，并将其重定向到443端口
        connector.setScheme("http");
        connector.setPort(80);
        connector.setSecure(false);
        connector.setRedirectPort(443);
        return connector;
    }

    @Bean
    public TomcatServletWebServerFactory servletContainer() {
        // 对http请求添加安全性约束，将其转换为https请求
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        tomcat.addAdditionalTomcatConnectors(connector());
        return tomcat;
    }

}
```