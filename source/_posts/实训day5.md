---
title: 实训day5
date: 2021-06-4 00:00:00
# 文章出处名称 #
from: 
# 文章出处链接 #
url: 
# 文章作者名称 #
author: 老叭美食家
# 文章作者签名 #
about: 
# 文章作者头像 #
avatar: 
# 是否开启评论 #
comments: false
# 文章标签 #
tags: 实训
# 文章分类 #
categories: 国信安实训
# 文章摘要 #
description: Spring入门4
# 文章置顶 #
sticky: 
---

# Spring入门4

项目代码:https://gitee.com/laobameishijia/guoxinan-practical-training

- [Spring入门4](#spring入门4)
  - [简单登录页面实现](#简单登录页面实现)
    - [预期效果](#预期效果)
    - [实现思路](#实现思路)
    - [实现过程](#实现过程)
      - [创建服务接口，并实现](#创建服务接口并实现)
      - [写好mapper,进行数据查询](#写好mapper进行数据查询)
      - [控制器注册服务，传参](#控制器注册服务传参)
  - [Springboot数据校验](#springboot数据校验)
    - [搞清楚什么是面向切面编程](#搞清楚什么是面向切面编程)
    - [实体类----写上相关注解](#实体类----写上相关注解)
    - [校验类----检验数据、抛出异常,由异常处理类进行处理](#校验类----检验数据抛出异常由异常处理类进行处理)
    - [控制器----将前端传入数据传给校验类进行校验](#控制器----将前端传入数据传给校验类进行校验)
  - [Springboot全局异常](#springboot全局异常)
    - [异常处理类](#异常处理类)
    - [自定义异常--以登录异常为例](#自定义异常--以登录异常为例)
    - [在服务中抛出异常](#在服务中抛出异常)
  - [Spring拦截器](#spring拦截器)
    - [登录拦截器](#登录拦截器)
    - [注册拦截器](#注册拦截器)

## 简单登录页面实现

### 预期效果

- 登录成功，进入主页—登录成功
- 登录失败：告诉用户具体错误--用户不存在、密码不正确、登录失败
- 退出登录，提示用户是否退出，点击是删除session退出登录。

### 实现思路

- 处理持久层—(操作数据库的Mapper)的代码：查询---findByName(String adminName) 找不到—提示用户不存在
- 处理登录业务逻辑(服务Service)----实现登录失败、密码不正确几种情况的逻辑。
- 控制层----路由控制、结果返回
- 表现层(视图、网页)----ajax异步请求、Session保留会话

![20210605102000](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210605102000.png)

### 实现过程

#### 创建服务接口，并实现

接口：

```java
public interface LoginService {

    /**
     * 用户登录
     * @param admin
     * @return
     */
    JsonData login (Admin admin, HttpSession session);
    JsonData exit (HttpServletRequest request);
}
```

实现：

```java
@Service
public class LoginServiceImpl implements LoginService {
    
    @Autowired
    private AdminMapper mapper;
    
    @Override
    public  JsonData login(Admin admin, HttpSession session){
        //数据校验
        
        //从数据库中查数据
        Admin dbAdmin = mapper.findoneByName(admin.getAdminName());
        
        if (dbAdmin == null) {
            //说明用户不存在
//            return new JsonData(1001,"用户不存在");
            throw new LoginException(1001,"用户不存在");
        }
        
        //判断状态
        if(dbAdmin.getAdminStatus()!=null && !(dbAdmin.getAdminStatus().equals(0))){
//            return new JsonData(1001,"用户被锁定,联系管理员");
            throw new LoginException(1002,"用户被锁定,联系管理员");
        }
        
        //判断密码正确
        if(!dbAdmin.getAdminPwd().equals(admin.getAdminPwd())){
//            return new JsonData(1001,"密码错误");
            throw new LoginException(1003,"密码错误");

        }

        //存到session
        session.setAttribute("adminName",admin.getAdminName());

        //更新用户最后登录时间
        dbAdmin.setLastLoginTime(new Timestamp(System.currentTimeMillis()));

        //dbAdmin没有问题
        System.out.println(dbAdmin.toString());

        mapper.update(dbAdmin);


        return new JsonData(200,"登录正常");
        
    }

    @Override
    public  JsonData exit(HttpServletRequest request){
        request.getSession().invalidate();
        return new JsonData(200,"退出正常");
    }
}
```



#### 写好mapper,进行数据查询

```java
/**
 * 主要用来操作数据库，增删改查
 */
@Mapper
public interface AdminMapper {

    /**
     * 查询所有数据
     */
    List<Admin> findAll();

    List<Admin> fineByParam(@Param("param") MyParam param);

    /**
     *增加
     * @param admin
     */
    void save(@Param("admin") Admin admin);

    /**
     * 真删除
     */
    void deleteTrue(@Param("delete") Integer id);

    /**
     * 更新
     * @param admin
     */
    void update(@Param("admin") Admin admin);

    /**
     * 软删除
     * @param id
     */
    void deleteFalse(@Param("id") Integer id);

    /**
     * 查询单条数据
     * @param id
     */
    Admin findone(@Param("id") Integer id);

    /**
     * 通过用户名查询对应的用户
     * @param name
     * @return
     */
    Admin findoneByName(@Param("name") String name);

    /**
     * 批量插入
     */
    int insertBatch(@Param("adminList") List<Admin> adminList);

    /**
     * 批量删除
     * @param adminList
     * @return
     */
    int deleteBatch(@Param("adminList") List<Admin> adminList);

}
```


#### 控制器注册服务，传参

```java
public class LoginController {

    @Autowired
    private LoginServiceImpl loginService;

    @RequestMapping("/page")
    public String login() {
        return "login";
    }

    @PostMapping("/verify")
    @ResponseBody//不需要进行页面跳转而是直接返回数据。
    //添加了@ResponseBody注解的方法，返回值会通过HTTP响应主体直接发送给浏览器。
    public JsonData verifyLogin(@Validated Admin admin, BindingResult result, HttpSession session) throws Exception {
        //数据校验
        //获取所有错误
        ValidatorUtil.showMsg(result);

        return loginService.login(admin, session);
    }

    @PostMapping("/exit")
    @ResponseBody//不需要进行页面跳转而是直接返回数据。
    //添加了@ResponseBody注解的方法，返回值会通过HTTP响应主体直接发送给浏览器。
    public JsonData exit(HttpServletRequest request) throws Exception {

        return loginService.exit(request);
    }
```

## Springboot数据校验

### 搞清楚什么是面向切面编程

AOP技术利用一种称为“横切”的技术，剖解开封装对象的内部，将影响多个类的公共行为封装到一个可重用的模块中，并将其命名为Aspect切面。所谓的切面，简单来说就是与业务无关，却为业务模块所共同调用的逻辑，将其封装起来便于减少系统的重复代码，降低模块的耦合度，有利用未来的可操作性和可维护性。

例如：银行系统的取款流程和查询余额的流程

![20210605095214](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210605095214.png)

hibernate validator 加几个注释，由后端检验
一般来说，Web应用都是前后端都会对数据进行校验，前端一般是用js正则进行校验，后端主要是对传入接口的数据进行校验，不能对一些无效的数据产生返回。

### 实体类----写上相关注解

```java
public class Admin {

  private Integer id;
  @NotBlank(message = "用户名不能为空！")
  private String adminName;
  @NotBlank(message = "密码不能为空")
  private String adminPwd;
  private Long adminPhone;
  private Timestamp lastLoginTime;
  private Timestamp createTime;
  private Timestamp updateTime;
  private Integer adminStatus;
  private Integer isDeleted;
}
```

### 校验类----检验数据、抛出异常,由异常处理类进行处理

```java
public class ValidatorUtil {

    /**
     * 展示错误信息
     */
    public static void showMsg(BindingResult result) throws Exception {
        List<ObjectError> allErrors = result.getAllErrors();

        for (ObjectError error : allErrors) {
            throw  new Exception(error.getDefaultMessage());
        }
    }
}
```

### 控制器----将前端传入数据传给校验类进行校验

```java
public class ValidatorUtil {

    /**
     * 展示错误信息
     */
    public static void showMsg(BindingResult result) throws Exception {
        List<ObjectError> allErrors = result.getAllErrors();

        for (ObjectError error : allErrors) {
            throw  new Exception(error.getDefaultMessage());
        }
    }
}
```







## Springboot全局异常

着重去理解异常类抛出和处理的顺序。抛出了哪个是由哪个类处理，往调用者抛出????

SpringBoot中有一个`ControllerAdvice`的注解，使用该注解表示开启了全局异常的捕获，我们只需在自定义一个方法使用`ExceptionHandler`注解然后定义捕获异常的类型即可对这些捕获的异常进行统一的处理。非常方便后续异常的分类处理以及代码维护

### 异常处理类

```java
@RestControllerAdvice
public class MyExceptionAdvice {
    
    /**
     * 专门用于处理登录异常
     * @param e
     * @return
     */
    @ExceptionHandler(LoginException.class)
    public JsonData loginExceptionHandler(LoginException e){
        return new JsonData(e.getCode(),e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    public JsonData  exceptionHandler(Exception e) {
        //记录异常日志
        //异常日志对于系统非常重要
        
        return new JsonData(1002,e.getMessage());
    }
}
```

### 自定义异常--以登录异常为例

```java
public class LoginException extends RuntimeException {

    private Integer code;

    private String msg;


    public LoginException() {
    }

    public LoginException(Integer code, String msg) {
        super(msg);
        this.code = code;
        this.msg = msg;
    }

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }
}
```

### 在服务中抛出异常

```java
@Service
public class LoginServiceImpl implements LoginService {
    
    @Autowired
    private AdminMapper mapper;
    
    @Override
    public  JsonData login(Admin admin, HttpSession session){
        //数据校验
        
        //从数据库中查数据
        Admin dbAdmin = mapper.findoneByName(admin.getAdminName());
        
        if (dbAdmin == null) {
            //说明用户不存在
//            return new JsonData(1001,"用户不存在");
            throw new LoginException(1001,"用户不存在");
        }
        
        //判断状态
        if(dbAdmin.getAdminStatus()!=null && !(dbAdmin.getAdminStatus().equals(0))){
//            return new JsonData(1001,"用户被锁定,联系管理员");
            throw new LoginException(1002,"用户被锁定,联系管理员");
        }
        
        //判断密码正确
        if(!dbAdmin.getAdminPwd().equals(admin.getAdminPwd())){
//            return new JsonData(1001,"密码错误");
            throw new LoginException(1003,"密码错误");

        }

        //存到session
        session.setAttribute("adminName",admin.getAdminName());

        //更新用户最后登录时间
        dbAdmin.setLastLoginTime(new Timestamp(System.currentTimeMillis()));

        //dbAdmin没有问题
        System.out.println(dbAdmin.toString());

        mapper.update(dbAdmin);


        return new JsonData(200,"登录正常");
        
    }

    @Override
    public  JsonData exit(HttpServletRequest request){
        request.getSession().invalidate();
        return new JsonData(200,"退出正常");
    }


}
```







## Spring拦截器

应用的例子：在用户没有登录的时候，无法进入系统中的其他页面。

原理：
对每一个请求进行审查，如果满足要求，则放行；不满足要求，重定向到其他页面。
**需要注意的是，要严格审查逻辑，放行登录页面和静态资源，不要产生无限循环的情况。**

![20210605103735](https://laoba-1304292449.cos.ap-chengdu.myqcloud.com/img/20210605103735.png)

### 登录拦截器

```java
/**
 * 登录拦截器
 */
public class LoginInterceptor implements HandlerInterceptor {

    /**
     * 前置方法
     * @param request
     * @param response
     * @param handler
     * @return
     * @throws Exception
     */
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        //判断是否登录
        //判断session
        //没有登录重定向到登录页面、登录了，定向到index页面

        //先去获取session对象
        HttpSession session = request.getSession();

        //获取登录的标记
        String adminName = (String) session.getAttribute("adminName");
        //判断session的值是否为null
        if (adminName == null) {
            //如果没有登录，这里就产生了循环，因为默认是拦截所有请求，所以就变成了无限次的重定向，
            //浏览器出现了too many redirect
            response.sendRedirect("/login/page");
            return false;
        }
        return true;//false拦截、true放行
    }

    /**
     * 后置方法
     * @param request
     * @param response
     * @param handler
     * @param modelAndView
     * @throws Exception
     */
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {

    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {

    }

}
```

### 注册拦截器

```java
@Configuration
public class SystemConfig implements WebMvcConfigurer {
    /**
     * 专门用来注册拦截器的
     * @param registry
     */
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LoginInterceptor())
                //拦截所有的请求
                .addPathPatterns("/**")
                //放行方法
                .excludePathPatterns("/login/**")
                //放行css
                .excludePathPatterns("/static/**");
    }
}
```