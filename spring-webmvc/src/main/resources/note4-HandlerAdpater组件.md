

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
            | |           
   RequestMappingHandlerAdapter     
             ^                     
            | |   implements BeanFactoryAware, InitializingBean 
            | |  afterPropertiesSet(), supportsInternal()  ,  handleInternal(),  getLastModifiedInternal()      
            | |       
   

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



##### 3. RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter implements BeanFactoryAware, InitializingBean 

   ###### 3.1) 属性      

   ```
   	private List<HandlerMethodArgumentResolver> customArgumentResolvers;
   
   	private HandlerMethodArgumentResolverComposite argumentResolvers;  // 解析方法参数的组件 
   
   	private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;  // 初始化数据的组件
   
   	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
   
   	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;  // 解析返回值的组件      
   
   	private List<ModelAndViewResolver> modelAndViewResolvers;  // 解析modelAndView的组件         
   
   	private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();
   
   	private List<HttpMessageConverter<?>> messageConverters; // 消息转换器      
   
   	private List<Object> requestResponseBodyAdvice = new ArrayList<Object>();
   
   	private WebBindingInitializer webBindingInitializer;  // web绑定初始器   
   ```


   ###### 3.2)  初始化      
   ```
   @Override
   	public void afterPropertiesSet() {
   		// Do this first, it may add ResponseBody advice beans
   		initControllerAdviceCache();
   
        // 初始化方法参数解析器   
   		if (this.argumentResolvers == null) {
   		   // 获取默认的参数解析器 
   			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers(); 
   			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   		}
   		
   		if (this.initBinderArgumentResolvers == null) {
   			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
   			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
   		}
   		
   		// 初始化处理器方法返回值解析器  
   		if (this.returnValueHandlers == null) {
   		    // 获取默认的返回值解析器   
   			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
   			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
   		}
   	}
   ```
   
   
   ```
   /**
   	 * Return the list of argument resolvers to use including built-in resolvers
   	 * and custom resolvers provided via {@link #setCustomArgumentResolvers}.
   	 */
   	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
   		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<HandlerMethodArgumentResolver>();
   
   		// Annotation-based argument resolution
   		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
   		resolvers.add(new RequestParamMapMethodArgumentResolver());
   		resolvers.add(new PathVariableMethodArgumentResolver());
   		resolvers.add(new PathVariableMapMethodArgumentResolver());
   		resolvers.add(new MatrixVariableMethodArgumentResolver());
   		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
   		resolvers.add(new ServletModelAttributeMethodProcessor(false));
   		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
   		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
   		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
   		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
   		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
   		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
   		resolvers.add(new SessionAttributeMethodArgumentResolver());
   		resolvers.add(new RequestAttributeMethodArgumentResolver());
   
   		// Type-based argument resolution
   		resolvers.add(new ServletRequestMethodArgumentResolver());
   		resolvers.add(new ServletResponseMethodArgumentResolver());
   		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
   		resolvers.add(new RedirectAttributesMethodArgumentResolver());
   		resolvers.add(new ModelMethodProcessor());
   		resolvers.add(new MapMethodProcessor());
   		resolvers.add(new ErrorsMethodArgumentResolver());
   		resolvers.add(new SessionStatusMethodArgumentResolver());
   		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
   
   		// Custom arguments
   		if (getCustomArgumentResolvers() != null) {
   			resolvers.addAll(getCustomArgumentResolvers());
   		}
   
   		// Catch-all
   		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
   		resolvers.add(new ServletModelAttributeMethodProcessor(true));
   
   		return resolvers;
   	}
   ```
   
   ```
   /**
   	 * Return the list of return value handlers to use including built-in and
   	 * custom handlers provided via {@link #setReturnValueHandlers}.
   	 */
   	private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
   		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<HandlerMethodReturnValueHandler>();
   
   		// Single-purpose return value types
   		handlers.add(new ModelAndViewMethodReturnValueHandler());
   		handlers.add(new ModelMethodProcessor());
   		handlers.add(new ViewMethodReturnValueHandler());
   		
   		// 可以通过setMessageConverters()来设置, 一般用来配置json     
   		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters()));   
   		handlers.add(new StreamingResponseBodyReturnValueHandler());
   		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),
   				this.contentNegotiationManager, this.requestResponseBodyAdvice));
   		handlers.add(new HttpHeadersReturnValueHandler());
   		handlers.add(new CallableMethodReturnValueHandler());
   		handlers.add(new DeferredResultMethodReturnValueHandler());
   		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));
   
   		// Annotation-based return value types
   		handlers.add(new ModelAttributeMethodProcessor(false));
   		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
   				this.contentNegotiationManager, this.requestResponseBodyAdvice));
   
   		// Multi-purpose return value types
   		handlers.add(new ViewNameMethodReturnValueHandler());
   		handlers.add(new MapMethodProcessor());
   
   		// Custom return value types
   		if (getCustomReturnValueHandlers() != null) {
   			handlers.addAll(getCustomReturnValueHandlers());
   		}
   
   		// Catch-all
   		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
   			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
   		}
   		else {
   			handlers.add(new ModelAttributeMethodProcessor(true));
   		}
   
   		return handlers;
   	}
   ```
   
   ###### 3.3)  实现父类方法 
   ```
   /**
   	 * Always return {@code true} since any method argument and return value
   	 * type will be processed in some way. A method argument not recognized
   	 * by any HandlerMethodArgumentResolver is interpreted as a request parameter
   	 * if it is a simple type, or as a model attribute otherwise. A return value
   	 * not recognized by any HandlerMethodReturnValueHandler will be interpreted
   	 * as a model attribute.
   	 */
   	@Override
   	protected boolean supportsInternal(HandlerMethod handlerMethod) {
   		return true;
   	}
   ``` 
   
   
   
   ```
   // 这里主要是处理了同步异步, 实际调用处理方法委托给了invokeHandlerMethod()   
   @Override
   	protected ModelAndView handleInternal(HttpServletRequest request,
   			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
   
   		ModelAndView mav;
   		checkRequest(request);
   
   		// Execute invokeHandlerMethod in synchronized block if required.
   		if (this.synchronizeOnSession) {
   			HttpSession session = request.getSession(false);
   			if (session != null) {
   				Object mutex = WebUtils.getSessionMutex(session);
   				synchronized (mutex) {
   					mav = invokeHandlerMethod(request, response, handlerMethod);
   				}
   			}
   			else {
   				// No HttpSession available -> no mutex necessary
   				mav = invokeHandlerMethod(request, response, handlerMethod);
   			}
   		}
   		else {
   			// No synchronization on session demanded at all...
   			mav = invokeHandlerMethod(request, response, handlerMethod);
   		}
   
   		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
   			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
   				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
   			}
   			else {
   				prepareResponse(response);
   			}
   		}
   
   		return mav;
   	}
   ```
   
   ```
   /**
   	 * Invoke the {@link RequestMapping} handler method preparing a {@link ModelAndView}
   	 * if view resolution is required.
   	 * @since 4.2
   	 * @see #createInvocableHandlerMethod(HandlerMethod)
   	 */
   	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
   			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
   
   		ServletWebRequest webRequest = new ServletWebRequest(request, response);
   		try {
   			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
   			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
   
            // 创建invocableHandlerMethod   
   			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
   			// 设置参数解析器  
   			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);  
   			// 设置参数返回值解析器 
   			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
   			invocableMethod.setDataBinderFactory(binderFactory);
   			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
   
            // 创建modelAndView容器   
   			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
   			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
   			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
   			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
   
   			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
   			asyncWebRequest.setTimeout(this.asyncRequestTimeout);
   
   			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
   			asyncManager.setTaskExecutor(this.taskExecutor);
   			asyncManager.setAsyncWebRequest(asyncWebRequest);
   			asyncManager.registerCallableInterceptors(this.callableInterceptors);
   			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
   
   			if (asyncManager.hasConcurrentResult()) {
   				Object result = asyncManager.getConcurrentResult();
   				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
   				asyncManager.clearConcurrentResult();
   				if (logger.isDebugEnabled()) {
   					logger.debug("Found concurrent result value [" + result + "]");
   				}
   				invocableMethod = invocableMethod.wrapConcurrentResult(result);
   			}
   
            // 实际调用处理, 最后将model存放于ModelAndViewContainer中  
   			invocableMethod.invokeAndHandle(webRequest, mavContainer);
   			if (asyncManager.isConcurrentHandlingStarted()) {
   				return null;
   			}
            
   			return getModelAndView(mavContainer, modelFactory, webRequest);
   		}
   		finally {
   			webRequest.requestCompleted();
   		}
   	}
   ```
   
   ServletInvocableHandlerMethod#invokeAndHandle()
   ```
   /**
   	 * Invoke the method and handle the return value through one of the
   	 * configured {@link HandlerMethodReturnValueHandler}s.
   	 * @param webRequest the current request
   	 * @param mavContainer the ModelAndViewContainer for this request
   	 * @param providedArgs "given" arguments matched by type (not resolved)
   	 */
   	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
   			Object... providedArgs) throws Exception {
   
         // 调用处理请求    
         // 调用参数解析器,解析参数, 然后使用反射调用对应的方法 
   		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
   		setResponseStatus(webRequest);
   
   		if (returnValue == null) {
   			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
   				mavContainer.setRequestHandled(true);
   				return;
   			}
   		}
   		else if (StringUtils.hasText(getResponseStatusReason())) {
   			mavContainer.setRequestHandled(true);
   			return;
   		}
   
   		mavContainer.setRequestHandled(false);
   		try {
   		    // 这里调用返回值解析器处理返回值  
   			this.returnValueHandlers.handleReturnValue(
   					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
   		}
   		catch (Exception ex) {
   			if (logger.isTraceEnabled()) {
   				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
   			}
   			throw ex;
   		
   ```





