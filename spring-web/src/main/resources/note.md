

### notebook for spring-mvc

#### 1. 上下文的初始化

   spring mvc存在两个上下文, 一个是root 上下文, 另一个是servlet上下文, servlet上下文的父上下文是root上下文.
   servlet上下文包含了controllers, view resolvers等mvc中需要的组件, root上下文则包含了
   services, repositories 
   
   ![spring-mvc contexts 图](./mvc-contexts.png)
   
   ```
    <!--配置springmvc DispatcherServlet-->
    <servlet>
        <servlet-name>springMVC</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <!--配置dispatcher.xml作为mvc的配置文件-->
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/dispatcher-servlet.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
        <async-supported>true</async-supported>
    </servlet>
    <servlet-mapping>
        <servlet-name>springMVC</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
    <!--把applicationContext.xml加入到配置文件中-->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
 ```

   上面的配置是web.xml中的配置, root上下文由 org.springframework.web.context.ContextLoaderListener 
   监听器进行加载初始化, <listener/>符合servlet规范, 该监听器会随着web容器启动web应用而调用. 
   
   Servlet上下文 由  org.springframework.web.servlet.DispatcherServlet 自身进行初始化, 
   dispatcherServlet和其他mvc框架一样, 也是基于前端控制器模式(the front controller pattern), 
   将dispatcherServlet当做中心servlet, 它将会监听所有请求, 并分发请求到对应的controller处理
   
   
   
#### 2. root 上下文的初始化
   
    
    



