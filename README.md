# SSO-demo

## 1. 创建git项目

  ```
  echo "# SSO-demo" >> README.md
  git init
  git add README.md
  git commit -m "first commit"
  git remote add origin https://github.com/CarrotWang/SSO-demo.git
  git push -u origin master
  ```
  https://www.cnblogs.com/yabin/p/6366151.html (github markdown语法)

## 2. 创建maven项目
  
  我的方式是，在Idea里创建一个新的maven项目，然后加入spring boot依赖，以及你需要加载的模块的spring boot依赖（比如mybatis）；在开发过程中，遇到了新的依赖，都在maven的pom文件中接着添加
  ```
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
            <version>2.0.0.RELEASE</version>
        </dependency>
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.11</version>
        </dependency>
    </dependencies>
  ```
  
  ## 3. 基本代码编写
  + 添加java包，创建Main.java文件
  + Main类上加上Spring Boot的核心注解 @SpringBootApplication，编写Spring boot启动代码
  ```
    public static void main(String[] args) {
        SpringApplication.run(Main.class);
    }
  ```
  + 创建最常见的package：bean、controller、service、mapper、util等
  
  ## 4. Mybatis在Spring Boot中的使用
  + resources下创建application.properties文件，并配置数据源基本信息(所有配置信息都在这里)
  (创建数据库时记得编码设置为utf8-mb4)
```
  spring.datasource.url=jdbc:mysql://127.0.0.1:3306/sso
  spring.datasource.username=root
  spring.datasource.password=root
```
  + 在resources下创建mapper文件夹，并且在Main类上加上注解，指明mapper文件存放位置
```
  @MapperScan("carrot.mapper")
```
  我偷懒，不想去生存mapper文件，全是在mapper接口中直接写的sql。
  
  ## 5. 实现最简单的登录逻辑
  + 数据库表创建语句
  ```
  CREATE TABLE `user` (
    `id` bigint(11) unsigned NOT NULL AUTO_INCREMENT,
    `name` varchar(120) NOT NULL DEFAULT '' COMMENT 'name',
    `passwd` varchar(120) NOT NULL DEFAULT '' COMMENT 'password',
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
  ```
  + 创建dto包，用于存放传送数据的对象（将业务逻辑和bean解耦），创建LoginRequest类，用于发送登录请求
  + 编写注册接口
  + 编写登录接口
  + 用jquery和bootstrap写了一页面用于调试、展示
  
  ## 6. 登录业务逻辑实现思路
  ### 1.实现session
   + 连接redis
   
 ```
        <!-- 操作redis -->
        <dependency>
            <groupId>redis.clients</groupId>
            <artifactId>jedis</artifactId>
            <version>2.9.0</version>
        </dependency>
 ```
    
 新增redis配置，创建redis连接池（JedisClient类配置），在UserService处使用，
 也就是密码验证通过的时候，设置key-value，key为用户id，value暂且设置无意义字符串，后续使用。

```
jedisPool.getResource().setex("session_"+u.getId() , LOGIN_TIMEOUT_SECOND,"Session Object Json String" );
```

 ### 2.实现filter拦截
 #### 1.登录时激活cookie
      Cookie cookie = new Cookie(ConstantUtil.USER_SESSION_KEY, URLEncoder.encode(Long.toString(u.getId()), "UTF-8"));
      cookie.setMaxAge(LOGIN_TIMEOUT_SECOND);
      cookie.setPath("/");
      response.addCookie(cookie);
 #### 2.配置filter
      Main类上加注解@ServletComponentScan，激活servlet配置
      filter实现类上加注解@WebFilter
 #### 3.实现filter逻辑
      1. 过滤不拦截的资源
      2. 通过cookie检查用户是否登录，登录则更新session存活时间，接着继续处理；
         未登录，跳转到登录页面
      
  ## 7.单点登录原理
  ### 1.原理（CAS，Central Authentication Service）
  ![](https://github.com/CarrotWang/SSO-demo/blob/master/img/cas.jpg)
  
      1.登录普通网站A
      2.重定向到CAS登录页面
      3.用户输入用户名、密码，CAS服务进行验证
      4.验证通过，CAS让客户端重定向到网站A，并传入service名以及token
      5.服务A得到service和token后，请求CAS服务验证真实性
      6.CAS确定服务A请求的真实性后，返回该用户信息
  
  ### 2.简化版
  
  借助RSA算法简化上面的流程，同时保证信息安全性。
  
      1.登录普通网站A（A应用事先在CAS服务上注册，获得专有的RSA算法私钥）
      2.重定向到CAS登录页面
      3.用户输入用户名、密码，CAS服务进行验证
      4.验证通过，CAS让客户端重定向到网站A（一个Get方法接口），并传入token(由RSA算法加密的用户信息)
      5.A应用得到token后，使用私钥对其进行解密，得到用户信息将其存入session
      
  
  ### 3.实现
  由于前期考虑不周，这里修改原来的单模块项目为多模块项目，包含两个模块，server和client。
  server模块端即CAS服务器，client模块单独打包，其他服务需要单点服务时，依赖此模块，即可实现SSO功能。
    
  #### 1.filter实现对资源的拦截
  和之前的实现类似，通用资源不做拦截；检查session确定登录状态，不过这里重定向的页面不再是本站登录页面，而是统一认证服务登录页面。

  #### 2.实现token处理接口
  提供一个Get接口，CAS服务验证通过后，通知浏览器重定向到该接口，并携带包含用户信息的token。
  
  
  ## 6. 最终的项目如何跑起来？
      为了验证单点登录，我们至少要跑三个服务，
        1.统一登陆服务
        2.服务A1 
        3.服务A2 为了避免端口冲突 加上JVM参数 -Dserver.port=8098
  
  ## 7. 关联问题记后续改进  
  ### 1.各种加密算法以及HTTPS协议
      
  ### 2.遇到的两个小问题
        （1）token传输的编码问题
        （2）jedisPool使用不当导致的阻塞问题
  ### 3.遗留的一个小bug    
        logout这里有一个bug，其实懂了原理这里很好修改，算是一个作业。
  ### 4.可以继续优化的地方
        可以注意到这个项目分了三个package，common包就是为了方便其他服务使用；
        我们学了Spring Boot，它能够帮助自动加载需要的组件，我们可以把client包做成一个可配置的starter供介入CAS服务的客户使用。
