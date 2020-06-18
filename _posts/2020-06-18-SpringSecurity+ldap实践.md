---
title: 使用springsecurity接入ldap认证
published: true
---

# 基础使用

首先加入相关依赖

```xml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-ldap</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-ldap</artifactId>
            <version>5.2.1.RELEASE</version>
        </dependency>

```

新建一个配置类

```java
@EnableWebSecurity
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.ldapAuthentication()
                .userDnPatterns("uid={0},ou=people,dc=rd,dc=netease,dc=com")
                .contextSource()
                .url("this is your ldap server host")
                .and()
                .passwordCompare()
                .passwordEncoder(null)
                .passwordAttribute("userPassword");
    }


    @Override
    protected void configure(HttpSecurity http) throws Exception {
        //这里可以配置鉴权行为
        
    }

    @Override
    public void configure(WebSecurity web) throws Exception {
        //前台接口不需要
        web.ignoring().antMatchers("/api/v1/pub/**");
    }
}
```

**注意：因为springsecurity默认是有编码的。如果ldap服务器使用明文的方式传递password，这里对编码方式passwordEncoder(null)需要置空，否则password匹配失败，ldapcode为50**

配置类的内容：

- configure(AuthenticationManagerBuilder auth)方法，定义了一些基础的配置：服务器host，port，密码的命名，ou，dc等等
- configure(HttpSecurity http)，定义鉴权行为，例如鉴权的接口，失败后的行为，登出的接口等
- configure(WebSecurity web)，定义资源的管理，哪些需要哪些不需要，或者更细粒度的控制

# security+ldap默认流程

1. 浏览器发起请求
2. 服务器拿到session数据
3. 判断数据中是否存在token，有转【4】，无转【5】
4. 获取数据，结束
5. 响应302，重定向至login页面，带原目标url
6. 用户输入用户名和密码提交，请求login接口，带原目标url
7. 后台获取到用户名和密码，请求ldap服务器
8. 若成功转【9】，失败转【10】
9. 将token放入response，重定向至目标url，结束
10. 转【5】

# 自定义鉴权失败行为

默认情况下，请求接口时如果没有带token，会返回 `httpcode:302`  后重定向至login页面（springsecurity提供的一个简洁的页面），但这个页面是后台服务提供的，不符合前后端分离的场景，所以需要进行配置，如下：

```java

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.exceptionHandling().authenticationEntryPoint( new AuthenticationEntryPoint(){
            @Override
            public void commence(HttpServletRequest req, HttpServletResponse res, AuthenticationException e) throws IOException, ServletException {
                res.setContentType("application/json;charset=utf-8");
                PrintWriter writer = res.getWriter();
                //todo 这里进行返回body的设置
            }
        });
    }

```

