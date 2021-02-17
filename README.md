# session一致性

## 开发工具

- 基于WSL2(适用于 Linux 的 Windows 子系统)的Ubuntu20.x
- Tomcat
- Redis
- Nginx
- 基于springboot的sessionProject

## 开发步骤

- java代码基于老师提供loginproject（redis存储session）

  - 首先启动类上，需要加上注释`@EnableRedisHttpSession`

    ```java
    @SpringBootApplication
    @EnableCaching
    @EnableRedisHttpSession
    public class LoginApplication  extends SpringBootServletInitializer {
    
        @Override
        protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
            return builder.sources(LoginApplication.class);
        }
    
        public static void main(String[] args) {
            SpringApplication.run(LoginApplication.class, args);
        }
    
    }
    ```

  - 修改配置文件

    ```properties
    spring.http.encoding.force-response=true
    
    #jsp 支持
    spring.mvc.view.suffix=.jsp
    spring.mvc.view.prefix=/WEB-INF/jsp/
    
    #关闭默认模板引擎
    spring.thymeleaf.cache=false
    spring.thymeleaf.enabled=false
    
    spring.redis.host=localhost
    spring.redis.port=6379
    redis.connectionTimeout=5000
    spring.redis.password=
    spring.redis.database=0
    ```

  - 修改pom文件，让项目可以打成war包

    ```xml
    <packaging>war</packaging>
    
    <properties>
        <java.version>8</java.version>
    </properties>
    ```

打包出来的war包一式两份，用于部署分布式服务

- Tomcat修改配置文件

  Tomcat默认的部署文件夹为webapps，在同一层级下，我们新建一个prod文件夹，把打包好的war包放进去，目录结构为

  ```
  Tomcat
  ├── prod
  │   ├── LoginProject.war
  ├── webapps
  │   ├── LoginProject.war
  ```

  - 修改Tomcat的server.xml配置文件

    在原来的<service>标签的基础上，再新增一对<service>标签name为**Catalina2**，<Connector>标签的 pord属性改为8081，<host>标签下新增<Context>标签

    - Catalina <host> appBase指向webapps
    - Catalina2 <host> appBase指向prod

    具体配置如下：

  ```xml
    <Service name="Catalina">
  
      <Connector port="8080" protocol="HTTP/1.1"
                 connectionTimeout="20000"
                 redirectPort="8443" />
      <Engine name="Catalina" defaultHost="localhost">
  
        <Realm className="org.apache.catalina.realm.LockOutRealm">
         
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
  
        <Host name="localhost"  appBase="webapps"
              unpackWARs="true" autoDeploy="true">
  		<Context path="" docBase="LoginProject.war" debug="0" privileged="true"></Context>
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log" suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
  
        </Host>
      </Engine>
    </Service>
  
    <Service name="Catalina2">
      <Connector port="8081" protocol="HTTP/1.1"
                 connectionTimeout="20000"
                 redirectPort="8443" />
      <Engine name="Catalina2" defaultHost="localhost">
  
        <Realm className="org.apache.catalina.realm.LockOutRealm">
          <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
                 resourceName="UserDatabase"/>
        </Realm>
  
        <Host name="localhost"  appBase="prod"
              unpackWARs="true" autoDeploy="true">
  
  		<Context path="" docBase="LoginProject.war" debug="0" privileged="true"></Context>
  
          <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
                 prefix="localhost_access_log" suffix=".txt"
                 pattern="%h %l %u %t &quot;%r&quot; %s %b" />
  
        </Host>
      </Engine>
    </Service>
  ```

  这样一个tomcat下就能跑两个service了，端口分别是8080和8081

- 修改Nginx配置文件

  修改nginx.conf配置文件，具体配置如下：

  ```
  	#负载均衡，轮询方式
  	upstream  sessionProject {
   	      server    localhost:8080;
   	      server    localhost:8081;
  	}
  	#监听8888端口，转发到sessionProject下的服务
  	server {
              listen       8888;
              server_name  localhost;
  
              location / {
                      proxy_pass http://sessionProject;
                      proxy_redirect default;
              }
  
      	}
  ```

## 实现效果

- 访问http://localhost:8888/能路由到toLogin页面

- 刷新toLogin页面时，能交替切换到8080和8081服务
- 登陆后刷新result页面，能交替切换到8080和8081服务，且不回跳到登录页

**具体的演示在项目的演示视频中**

**nginx以及tomcat的具体配置文件在conf文件夹下**



