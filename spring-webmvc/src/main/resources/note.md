

#### dispatcher servlet 上下文的加载
   
   HttpServletBean  extends HttpServlet implements EnvironmentCapable, EnvironmentAware  
         ^  
        | |  init()   使用BeanWrapper将httpServlet封装成Bean, 设置bean属性, 留有initServletBean()供子类实现, 类似模板模式中的基本方法
        | |  setEnvironment(Environment environment) , ConfigurableEnvironment getEnvironment()    
   
   FrameworkServlet  implements ApplicationContextAware    
         ^   
        | | initServletBean()  创建servlet上下文  this.webApplicationContext = initWebApplicationContext();    
        | | initFrameworkServlet() 供给子类实现
        | |   
        | | 实现servlet中的doGet(), doPost()等方法, 交由processRequest(servletRequest, servletResponse)方法处理
        | | processRequest()方法中存在模板方法doService(request, response),  供给子类实现  
      
   DispatcherServlet     
        | |      
        | |  onRefresh(ApplicationContext context)   
        | |  initStrategies(ApplicationContext context)  初始化9大组件 
        | |     
        | |  doService()用于精准暴露request属性, 委派doDispatch()去做实际的分发工作
        | |   
   


