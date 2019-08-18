

#### HandlerAdapter组件

   ![HandlerAdapter组件类图](./img/HandlerAdapter组件类图.png)

##### 1. HandlerAdapter类图  

   HandlerAdapter 
            ^     
            |   boolean supports(Object handler);  // 一般使用instanceof 判断是否是某个类型的处理器  
            |   // 转换成Object类型的handler为对应实际类型, 调用对应方法处理, 返回modelAndView               
            |   ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;       
            |   long getLastModified(HttpServletRequest request, Object handler);  // 返回最后修改的时间     
            |  ---------------------------------------------------------------------- |          
            |                                                                         |                                    
   AbstractHandlerMethodAdapter                                      SimpleServletHandlerAdapter,                                                     
             ^     
            | |     
            | |       
   RequestMappingHandlerAdapter      
   

##### 2. AbstractHandlerMethodAdapter   extends WebContentGenerator implements HandlerAdapter, Ordered                 

  ######   2.1)  实现接口方法
  
  AbstractHandlerMethodAdapter#supports()     
  ```
  /**
  	 * This implementation expects the handler to be an {@link HandlerMethod}.
  	 * @param handler the handler instance to check
  	 * @return whether or not this adapter can adapt the given handler
  	 */
  	@Override
  	public final boolean supports(Object handler) {
  		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
  	}
  ```   
  
  AbstractHandlerMethodAdapter#handle()
  ```
  /**
     * Given a handler method, return whether or not this adapter can support it.
     * @param handlerMethod the handler method to check
     * @return whether or not this adapter can adapt the given method
     */
    protected abstract boolean supportsInternal(HandlerMethod handlerMethod);  // 用于判断是否支持
    
  ```
  
  
  ```
  	/**
  	 * This implementation expects the handler to be an {@link HandlerMethod}.
  	 */
  	@Override
  	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
  			throws Exception {
  
  		return handleInternal(request, response, (HandlerMethod) handler);
  	}
  	
  	/**
    	 * Use the given handler method to handle the request.
    	 * @param request current HTTP request
    	 * @param response current HTTP response
    	 * @param handlerMethod handler method to use. This object must have previously been passed to the
    	 * {@link #supportsInternal(HandlerMethod)} this interface, which must have returned {@code true}.
    	 * @return ModelAndView object with the name of the view and the required model data,
    	 * or {@code null} if the request has been handled directly
    	 * @throws Exception in case of errors
    	 */
    	protected abstract ModelAndView handleInternal(HttpServletRequest request,
    			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception;
  ``` 


  AbstractHandlerMethodAdapter#getLastModified()      
  ```
  /**
  	 * This implementation expects the handler to be an {@link HandlerMethod}.
  	 */
  	@Override
  	public final long getLastModified(HttpServletRequest request, Object handler) {
  		return getLastModifiedInternal(request, (HandlerMethod) handler);
  	}
  
  	/**
  	 * Same contract as for {@link javax.servlet.http.HttpServlet#getLastModified(HttpServletRequest)}.
  	 * @param request current HTTP request
  	 * @param handlerMethod handler method to use
  	 * @return the lastModified value for the given handler
  	 */
  	protected abstract long getLastModifiedInternal(HttpServletRequest request, HandlerMethod handlerMethod);

  ```













