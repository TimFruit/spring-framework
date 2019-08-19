

#### ViewResolver组件     

   ![ViewResolver组件类图](./img/ViewResolver组件类图.png)

##### 1. ViewResolver组件类图   

   ViewResolver
          ^     
          |    
          | View resolveViewName(String viewName, Locale locale) throws Exception;
          |      
          | ----------------------------------------------------------------------------|       
          |                                                                             |        
   AbstractCachingViewResolver                                                     ResourceBundleViewResolver(解决多国语言)                
          ^     
         | |    
         | |  实现了缓存功能, 用于缓存模板, 子类实现loadView()用于建立一个View对象           
         | |    
   UrlBasedViewResolver       
          ^      
         | |    
         | |  viewName是url的解决方案, redirect:myAction, forward:myAction        
         | |    
   AbstractTemplateViewResolver    
          ^      
         | |    
         | |  模板视图的基类解析器, 一般用于freemarker, velocity                      
         | |    
   FreeMarkerViewResolver        

















