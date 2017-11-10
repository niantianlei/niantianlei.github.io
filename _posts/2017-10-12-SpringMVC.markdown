---
layout:     post
title:      "SpringMVC学习整理"
subtitle:   " \"SpringMVC\""
date:   2017-10-12 10:04:18 +0800
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
catalog:    true
tags:
    - Java web开发
---



### 简介
Spring的MVC框架主要由DispatcherServlet、处理器映射、处理器(控制器)、视图解析器、视图组成。  

各种组件及其作用：
- **前端控制器(DispatcherServlet)：**接收请求，响应结果，相当于转发器，中央处理器。减少了其他组件之间的耦合度  
- **处理器映射器(HandlerMapping)：**能够完成客户请求到Controller映射  
- **控制器(Controller)：**需要为并发用户处理上述请求，因此实现Controller接口时，必须保证线程安全并且可重用。Controller将处理用户请求。一旦Controller处理完用户请求，则返回ModelAndView对象给DispatcherServlet前端控制器，ModelAndView中包含了模型（Model）和视图（View）。
从宏观角度考虑，DispatcherServlet是整个Web应用的控制器；从微观考虑，Controller是单个Http请求处理过程中的控制器，而ModelAndView是Http请求过程中返回的模型（Model）和视图（View）。  
- **处理器适配器(HandlerAdapter)：**按照特定规则(HandlerAdapter要求的规则)执行Controller   
- **视图解析器(ViewResolver)：**进行视图解析，根据逻辑视图解析成真正的视图(View)  
- **视图(View)：**View是一个接口实现类试吃不同的View类型（jsp,pdf等等）  

运行步骤：  
1.客户端请求提交到DispatcherServlet  
2.由DispatcherServlet控制器查询一个或多个HandlerMapping，找到处理请求的Controller  
3.DispatcherServlet将请求提交到Controller  
4.Controller调用业务逻辑处理后，返回ModelAndView(Springmvc框架的一个底层对象)  
5.DispatcherServlet查询一个或多个ViewResoler视图解析器，找到ModelAndView指定的视图  
6.视图解析器(ViewResolver)向前端控制器(DispatcherServlet)返回View  
7.前端控制器进行视图渲染，即将模型数据(在ModelAndView对象中)填充到request域  
8.前端控制器向用户响应结果  

实例：  
创建动态web项目，导入项目依赖的jar包。  
在WEB-INF目录下创建web.xml  
配置Spring MVC的入口DispatcherServlet，把所有的请求都提交到该Servlet  
```
<?xml version="1.0" encoding="UTF-8"?>
<web-app version="2.4" xmlns="http://java.sun.com/xml/ns/j2ee"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://java.sun.com/xml/ns/j2ee
http://java.sun.com/xml/ns/j2ee/web-app_2_4.xsd">
    <servlet>
        <servlet-name>springmvc</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <load-on-startup>1</load-on-startup>
        <!-- 若不配置springmvc加载的配置文件(配置处理器映射器、适配器等等)，
        默认加载WEB-INF/servlet名称-servlet(如springmvc-servlet.xml)-->
    </servlet>
    <servlet-mapping>
        <servlet-name>springmvc</servlet-name>
        <!-- /,所有访问的地址由DispatcherServlet进行解析，
        	对静态文件的解析需要配置不让DispatcherServlet进行解析，
           	 使用此种方式和实现RESTful风格的url -->
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
在WEB-INF目录下创建 springmvc-servlet.xml,其与上一步中的`<servlet-name>springmvc</servlet-name>`对应  
SpringMVC的映射配置文件  
```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE beans PUBLIC "-//SPRING//DTD BEAN//EN" "http://www.springframework.org/dtd/spring-beans.dtd">
<beans>
	<!-- 处理器映射器
    	将bean的name作为url进行查找，需要在配置Handler时指定beanname(就是url)
	 -->
    <bean id="simpleUrlHandlerMapping"
        class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping">
        <property name="mappings">
            <props>
                <prop key="/index">indexController</prop>
            </props>
        </property>
    </bean>
    <!-- 表示访问路径/index会交给id=indexController的bean处理 -->
    <bean id="indexController" class="controller.IndexController"></bean>
	
</beans>
```
控制类 IndexController实现接口Controller，提供方法handleRequest处理请求  
SpringMVC通过 ModelAndView 对象把模型和视图结合在一起  
```
package controller;
 
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
 
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.mvc.Controller;
 
public class IndexController implements Controller {
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ModelAndView mav = new ModelAndView("index.jsp");
        mav.addObject("message", "Hello Spring MVC");
        return mav;
    }
}
```
如上述代码所示：index.jsp表示视图，message表示数据，相当于(key，value)对。  
准备index.jsp，在WebContent中新建一个jsp文件，使用EL表达式显示message内容。
```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" isELIgnored="false"%>
 
<h1>${message}</h1>
```  
在Tomcat上运行，访问地址`http://localhost:8080/springmvc/index`。  
<br>
运行流程：  
1. 用户访问 /index  
2. 根据web.xml中的配置 所有的访问都会经过DispatcherServlet  
3. 根据配置文件springmvc-servlet.xml ，访问路径/index 会进入IndexController类   
4. 在IndexController中指定跳转到页面index.jsp，并传递message数据  
5. 在index.jsp中显示message信息  


<br />
### 视图定位
在`springmvc-servlet.xml`配置中增加如下代码：

```
<!-- 视图解析器  解析jsp,默认使用jstl,classpath下要有jstl的包 -->
<bean id="viewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
   <property name="prefix" value="/WEB-INF/page/" />
   <property name="suffix" value=".jsp" />
</bean>
```
作用：直接到/WEB-INF/page/下面去找jsp文件  
修改IndexController，将`ModelAndView mav = new ModelAndView("index.jsp");`中的index.jsp改为index。  
最后把index.jsp移动到 WEB-INF/page目录下。  
仍然访问地址`http://localhost:8080/springmvc/index`  

<br />  
### 注解方式
修改IndexController.java  
```
package controller;
 
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
 
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.servlet.ModelAndView;
 
@Controller
public class IndexController {
    @RequestMapping("/index")
    public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception {
        ModelAndView mav = new ModelAndView("index");
        mav.addObject("message", "Hello Spring MVC");
        return mav;
    }
}
```
类前面的注解`@Controller`表示这是一个控制器，在方法`handleRequest`前面加上`@RequestMapping("/index")`表示路径/index会映射到该方法上，此时无需实现Controller接口  
因为使用了注解方式，所以配置文件中的相关映射可以去除，增加`<context:component-scan base-package="controller" />`，表示从包controller里扫描控制器（即`@Controller`注解的类）  
改后如下：  
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:context="http://www.springframework.org/schema/context"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
    http://www.springframework.org/schema/context         
    http://www.springframework.org/schema/context/spring-context-3.0.xsd">
     
    <context:component-scan base-package="controller" />
    <!-- 视图解析器  解析jsp,默认使用jstl,classpath下要有jstl的包 -->
    <bean id="irViewResolver"
        class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/page/" />
        <property name="suffix" value=".jsp" />
    </bean>
<!--     <bean id="simpleUrlHandlerMapping" -->
<!--         class="org.springframework.web.servlet.handler.SimpleUrlHandlerMapping"> -->
<!--         <property name="mappings"> -->
<!--             <props> -->
<!--                 <prop key="/index">indexController</prop> -->
<!--             </props> -->
<!--         </property> -->
<!--     </bean> -->
<!--     <bean id="indexController" class="controller.IndexController"></bean> -->
</beans>
```

<br />

### 从表单获取数据
首先根据数据表创建实体类  
```
package pojo;
 
public class Product {
 
    private int id;
    private String name;
    private float price;
    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public float getPrice() {
        return price;
    }
    public void setPrice(float price) {
        this.price = price;
    }
     
}
```
在WebContent目录下,增加 增加商品的页面 addProduct.jsp  
```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" import="java.util.*" isELIgnored="false"%>
 
<form action="addProduct"> 
    产品名称 ：<input type="text" name="name" value=""><br />
    产品价格： <input type="text" name="price" value=""><br />
    <input type="submit" value="增加商品">
</form>
```
其中`isELIgnored="false"`表示该jsp是否忽视EL表达式，如果为true，那么JSP中的表达式被当成字符串处理  
默认为false  
准备控制器ProductController，提供一个add方法映射/addProduct路径。  
为add方法准备一个Product 参数，用于接收注入，最后跳转到showProduct页面显示用户提交的数据
```
package controller;
 
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.servlet.ModelAndView;
 
import pojo.Product;
 
@Controller
public class ProductController {
 
    @RequestMapping("/addProduct")
    public ModelAndView add(Product product) throws Exception {
        ModelAndView mav = new ModelAndView("showProduct");
        return mav;
    }
}
```
page目录下创建一个展示页面showProduct.jsp  
```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" isELIgnored="false"%>
 
产品名称： ${product.name}<br>
产品价格： ${product.price}
```
**说明：**from标签的action属性代表要提交到的地方，对应控制器的`@RequestMapping`。  


<br />

### 客户端跳转
在上一节介绍的表单数据获取，从addProduct 跳转到showProduct.jsp是服务端跳转，即url实际上不改变  
本节实现客户端跳转  
在IndexController中添加如下代码：  
```
@RequestMapping("/jump")
public ModelAndView jump() {
    ModelAndView mav = new ModelAndView("redirect:/index");
    return mav;
}  
```
访问`http://localhost:8080/springmvc/jump`，页面会重定向到`http://localhost:8080/springmvc/index`。  


<br />

### session
session一般用来传递数据  
修改IndexController，增加如下代码  
```
@RequestMapping("/check")
public ModelAndView check(HttpSession session) {
    Integer i = (Integer) session.getAttribute("count");
    if (i == null)
        i = 0;
    i++;
    session.setAttribute("count", i);
    ModelAndView mav = new ModelAndView("check");
    return mav;
}
```  
导入`import javax.servlet.http.HttpSession;`。  
在page目录下添加check.jsp  
```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" isELIgnored="false"%>
 
session中记录的访问次数：${count}
```
测试，访问`http://localhost:8080/springmvc/check`。  
刷新页面可观察到访问次数在增加。  

在IndexController中增加  
```
@RequestMapping("/clear")
public ModelAndView clear(HttpSession session) {
    session.setAttribute("count", -1);
    ModelAndView mav = new ModelAndView("redirect:/check");
    return mav;
}
```
访问路径/clear时，会清空session中的count，并且客户端重定向到/check。  

<br />

### 中文问题
如果addProduct.jsp中提交中文名的产品，观察地址栏`http://localhost:8080/springmvc/addProduct123?name=%E4%B8%AD%E6%96%87&price=123`。  
简单的通用方法是添加过滤器  
在web.xml中添加：  
```
<filter>  
    <filter-name>CharacterEncodingFilter</filter-name>  
    <filter-class>org.springframework.web.filter.CharacterEncodingFilter</filter-class>  
    <init-param>  
        <param-name>encoding</param-name>  
        <param-value>utf-8</param-value>  
    </init-param>  
</filter>  
<filter-mapping>  
    <filter-name>CharacterEncodingFilter</filter-name>  
    <url-pattern>/*</url-pattern>  
</filter-mapping>
```
然后改变addProduct.jsp中的提交方式method="post"。get方法为默认提交方法。  
查看地址栏：`http://localhost:8080/springmvc/addProduct123`。  
也就是说用post方法提交form表格，地址栏不会显示提交的数据。  


<br />

### 拦截器
新建类  
```
package IndexInterceptor;
import java.util.Date;
 
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
 
import org.springframework.web.servlet.ModelAndView;
import org.springframework.web.servlet.handler.HandlerInterceptorAdapter;
 
public class IndexInterceptor extends HandlerInterceptorAdapter {  
 
     /**  
     * 在业务处理器处理请求之前被调用  
     * 如果返回false  
     *     从当前的拦截器往回执行所有拦截器的afterCompletion(),再退出拦截器链 
     * 如果返回true  
     *    执行下一个拦截器,直到所有的拦截器都执行完毕  
     *    再执行被拦截的Controller  
     *    然后进入拦截器链,  
     *    从最后一个拦截器往回执行所有的postHandle()  
     *    接着再从最后一个拦截器往回执行所有的afterCompletion()  
     */   
    public boolean preHandle(HttpServletRequest request,    
            HttpServletResponse response, Object handler) throws Exception {
         
        System.out.println("preHandle(), 在访问Controller之前被调用");  
        return true;
         
    }  
 
    /** 
     * 在业务处理器处理请求执行完成后,生成视图之前执行的动作    
     * 可在modelAndView中加入数据，比如当前时间 
     */ 
     
    public void postHandle(HttpServletRequest request,    
            HttpServletResponse response, Object handler,    
            ModelAndView modelAndView) throws Exception {  
        System.out.println("postHandle(), 在访问Controller之后，访问视图之前被调用,这里可以注入一个时间到modelAndView中，用于后续视图显示");
        modelAndView.addObject("date","由拦截器生成的时间:" + new Date());
    }  
 
    /**  
     * 在DispatcherServlet完全处理完请求后被调用,可用于清理资源等   
     *   
     * 当有拦截器抛出异常时,会从当前拦截器往回执行所有的拦截器的afterCompletion()  
     */
     
    public void afterCompletion(HttpServletRequest request,    
            HttpServletResponse response, Object handler, Exception ex)  
    throws Exception {  
           
        System.out.println("afterCompletion(), 在访问视图之后被调用");  
    }  
       
} 
```
配置springmvc-servlet.xml，添加拦截  
```
<mvc:interceptors>    
    <mvc:interceptor>    
        <mvc:mapping path="/index"/>  
        <!-- 定义在mvc:IndexInterceptor下面的表示是对特定的请求才进行拦截的 --> 
        <bean class="IndexInterceptor.IndexInterceptor"/>      
    </mvc:interceptor>  
    <!-- 当设置多个拦截器时，先按顺序调用preHandle方法，然后逆序调用每个拦截器的postHandle和afterCompletion方法 --> 
</mvc:interceptors> 
```
该配置是对/index路径进行拦截，/\*\*拦截所有，/category/\*\* 拦截/category路径下的所有  
修改index.jsp，打印拦截器放进去的日期。  
```
<%@ page language="java" contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8" isELIgnored="false"%>
 
<h1>${message}</h1>
 
<p>${date}</p>
```
结果如下  
![a]({{ "/img/post/springmvc/1.png" | prepend: site.baseurl }} )  