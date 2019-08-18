

#### servlet上下文初始化 

##### 1. dispatcher servlet 类继承
   
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
   


##### 2. servlet上下文初始化   
    
   上面类图中， HttpServletBean则负责将Servlet规范中的HttpServlet封装成spring容器可管理的bean, FrameworkServlet则负责初始化
   servlet上下文， DispatcherServlet则负责初始化spring mvc 9大组件， 并负责分发http请求，将请求委托给各组件处理。
   
   
   2.1) HttpServletBean#init()
   ```
   /**
   	 * Map config parameters onto bean properties of this servlet, and
   	 * invoke subclass initialization.
   	 * @throws ServletException if bean properties are invalid (or required
   	 * properties are missing), or if subclass initialization fails.
   	 */
   	@Override
   	public final void init() throws ServletException {
   		if (logger.isDebugEnabled()) {
   			logger.debug("Initializing servlet '" + getServletName() + "'");
   		}
   
   		// Set bean properties from init parameters.
   		PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
   		if (!pvs.isEmpty()) {
   			try {
   				BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
   				ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
   				bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
   				initBeanWrapper(bw);
   				bw.setPropertyValues(pvs, true);
   			}
   			catch (BeansException ex) {
   				if (logger.isErrorEnabled()) {
   					logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
   				}
   				throw ex;
   			}
   		}
   
   		// Let subclasses do whatever initialization they like.
   		initServletBean();
   
   		if (logger.isDebugEnabled()) {
   			logger.debug("Servlet '" + getServletName() + "' configured successfully");
   		}
   	}
   ```
   
   2.2) FrameworkServlet#initServletBean()
   ```
   /**
   	 * Overridden method of {@link HttpServletBean}, invoked after any bean properties
   	 * have been set. Creates this servlet's WebApplicationContext.
   	 */
   	@Override
   	protected final void initServletBean() throws ServletException {
   		getServletContext().log("Initializing Spring FrameworkServlet '" + getServletName() + "'");
   		if (this.logger.isInfoEnabled()) {
   			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization started");
   		}
   		long startTime = System.currentTimeMillis();
   
   		try {
   			this.webApplicationContext = initWebApplicationContext();  //初始化上下文
   			initFrameworkServlet();  // 留给子类操作的方法
   		}
   		catch (ServletException ex) {
   			this.logger.error("Context initialization failed", ex);
   			throw ex;
   		}
   		catch (RuntimeException ex) {
   			this.logger.error("Context initialization failed", ex);
   			throw ex;
   		}
   
   		if (this.logger.isInfoEnabled()) {
   			long elapsedTime = System.currentTimeMillis() - startTime;
   			this.logger.info("FrameworkServlet '" + getServletName() + "': initialization completed in " +
   					elapsedTime + " ms");
   		}
   	}
   ```
   
   ```
   /**
   	 * Initialize and publish the WebApplicationContext for this servlet.
   	 * <p>Delegates to {@link #createWebApplicationContext} for actual creation
   	 * of the context. Can be overridden in subclasses.
   	 * @return the WebApplicationContext instance
   	 * @see #FrameworkServlet(WebApplicationContext)
   	 * @see #setContextClass
   	 * @see #setContextConfigLocation
   	 */
   	protected WebApplicationContext initWebApplicationContext() {
   	    
   	    // root上下文通过ContextLoaderListener监听器初始化了，它存放在ServletCotnext中
   	    // 通过工具类查找对应的root上下文
   		WebApplicationContext rootContext =
   				WebApplicationContextUtils.getWebApplicationContext(getServletContext());
   		WebApplicationContext wac = null;
   
        // 4种方法初始化servlet上下文 ==》 通过反射创建上下文实例， 然后设置Environment,configLocation等参数， 最后触发刷新
   		if (this.webApplicationContext != null) {  // (1) 上下文被注入
   			// A context instance was injected at construction time -> use it
   			wac = this.webApplicationContext;
   			if (wac instanceof ConfigurableWebApplicationContext) {
   				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
   				if (!cwac.isActive()) {
   					// The context has not yet been refreshed -> provide services such as
   					// setting the parent context, setting the application context id, etc
   					if (cwac.getParent() == null) {
   						// The context instance was injected without an explicit parent -> set
   						// the root application context (if any; may be null) as the parent
   						cwac.setParent(rootContext);
   					}
   					configureAndRefreshWebApplicationContext(cwac); // 配置刷新
   				}
   			}
   		}
   		if (wac == null) {  // 上下文没有被注入， 从servletContext中查找是否已经存在 
   			// No context instance was injected at construction time -> see if one
   			// has been registered in the servlet context. If one exists, it is assumed
   			// that the parent context (if any) has already been set and that the
   			// user has performed any initialization such as setting the context id
   			wac = findWebApplicationContext();
   		}
   		if (wac == null) {  // 没有上下文实例， 将创建上下文实例
   			// No context instance is defined for this servlet -> create a local one
   			wac = createWebApplicationContext(rootContext);
   		}
   
   		if (!this.refreshEventReceived) { // 触发刷新事件 
   			// Either the context is not a ConfigurableApplicationContext with refresh
   			// support or the context injected at construction time had already been
   			// refreshed -> trigger initial onRefresh manually here.
   			onRefresh(wac);
   		}
   
   		if (this.publishContext) {  // 发布上下文到servletContext
   			// Publish the context as a servlet context attribute.
   			String attrName = getServletContextAttributeName();
   			getServletContext().setAttribute(attrName, wac);
   			if (this.logger.isDebugEnabled()) {
   				this.logger.debug("Published WebApplicationContext of servlet '" + getServletName() +
   						"' as ServletContext attribute with name [" + attrName + "]");
   			}
   		}
   
   		return wac;
   	}
   ```
   
   DispatcherServlet#onRefresh()
   ```
   /**
   	 * This implementation calls {@link #initStrategies}.
   	 */
   	@Override
   	protected void onRefresh(ApplicationContext context) {
   		initStrategies(context);
   	}
   ``` 
   
   ```
   /**
   	 * Initialize the strategy objects that this servlet uses.
   	 * <p>May be overridden in subclasses in order to initialize further strategy objects.
   	 */
   	protected void initStrategies(ApplicationContext context) { //初始化9大组件
   		initMultipartResolver(context);
   		initLocaleResolver(context);
   		initThemeResolver(context);
   		initHandlerMappings(context);
   		initHandlerAdapters(context);
   		initHandlerExceptionResolvers(context);
   		initRequestToViewNameTranslator(context);
   		initViewResolvers(context);
   		initFlashMapManager(context);
   	}
   ```
   
   

#### dispatcherServlet分发流程 
   
   ##### 1. 分发原理
   web.xml中的配置如下， DispatcherServlet符合servlet规范， 能够为web容器识别， 通过配置， DispatcherServlet将负责监听所有
   http请求，并负责分发所有请求到对应的组件处理， 如Controllers
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
   ```
    
   FrameworkServlet实现了httpServlet的多种处理http请求的方法， 如doGet(), doPost()等方法， 最后委托给doService()方法处理
   DispatcherServlet继承了FrameworkServlet, 并实现了doService()方法， 分发流程委托给doDispatch()方法实现   
    
    
   ##### 2. 分发流程
   DispatcherServlet#doDispatch()
   ```
   /**
   	 * Process the actual dispatching to the handler.
   	 * <p>The handler will be obtained by applying the servlet's HandlerMappings in order.
   	 * The HandlerAdapter will be obtained by querying the servlet's installed HandlerAdapters
   	 * to find the first that supports the handler class.
   	 * <p>All HTTP methods are handled by this method. It's up to HandlerAdapters or handlers
   	 * themselves to decide which methods are acceptable.
   	 * @param request current HTTP request
   	 * @param response current HTTP response
   	 * @throws Exception in case of any kind of processing failure
   	 */
   	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   		HttpServletRequest processedRequest = request;
   		HandlerExecutionChain mappedHandler = null;
   		boolean multipartRequestParsed = false;
   
   		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
   
   		try {
   			ModelAndView mv = null;
   			Exception dispatchException = null;
   
   			try {
   			    //解析多部分请求 ， （一般是上传文件）
   				processedRequest = checkMultipart(request);
   				multipartRequestParsed = (processedRequest != request);  
   
                // 根据请求， 获取请求对应(映射)的mappedHandler 
                // mappedHandler封装了拦截器链
   				// Determine handler for the current request.
   				mappedHandler = getHandler(processedRequest);   
   				if (mappedHandler == null || mappedHandler.getHandler() == null) {
   					noHandlerFound(processedRequest, response);
   					return;
   				}
   
                // 获取对应的HandleAdpater, HandleAdpater对mappedHandler做了适配， 
                // 它用于适配不同类型的处理器，（可以是其他框架的处理器）， 然后调用执行对应的处理器，统一封装返回值ModelAndView
   				// Determine handler adapter for the current request.
   				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler()); 
   
   				// Process last-modified header, if supported by the handler.
   				String method = request.getMethod();
   				boolean isGet = "GET".equals(method);
   				if (isGet || "HEAD".equals(method)) {
   					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
   					if (logger.isDebugEnabled()) {
   						logger.debug("Last-Modified value for [" + getRequestUri(request) + "] is: " + lastModified);
   					}
   					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
   						return;
   					}
   				}
   
   				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
   					return;
   				}
   
                // 真正调用handlerAdapter处理结果
   				// Actually invoke the handler.
   				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());  // 通过handleAdapter处理请求
   
   				if (asyncManager.isConcurrentHandlingStarted()) {
   					return;
   				}
   
   				applyDefaultViewName(processedRequest, mv);
   				mappedHandler.applyPostHandle(processedRequest, response, mv);
   			}
   			catch (Exception ex) {
   				dispatchException = ex;
   			}
   			catch (Throwable err) {
   				// As of 4.3, we're processing Errors thrown from handler methods as well,
   				// making them available for @ExceptionHandler methods and other scenarios.
   				dispatchException = new NestedServletException("Handler dispatch failed", err);
   			}
   			
   			// 处理结果 ModelAndView, 如果结果是异常, 则封装异常,
   		    // 将ModleAndView转换成View, 最后由View渲染输出到response 
   			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);  // 处理输出返回值给请求方
   		}
   		catch (Exception ex) {
   			triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   		}
   		catch (Throwable err) {
   			triggerAfterCompletion(processedRequest, response, mappedHandler,
   					new NestedServletException("Handler processing failed", err));
   		}
   		finally {
   			if (asyncManager.isConcurrentHandlingStarted()) {
   				// Instead of postHandle and afterCompletion
   				if (mappedHandler != null) {
   					mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
   				}
   			}
   			else {
   				// Clean up any resources used by a multipart request.
   				if (multipartRequestParsed) {
   					cleanupMultipart(processedRequest);
   				}
   			}
   		}
   	}
   ```
    
  
  DispatcherServlet#processDispatchResult()  
  ```
  /**
  	 * Handle the result of handler selection and handler invocation, which is
  	 * either a ModelAndView or an Exception to be resolved to a ModelAndView.
  	 */
  	private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
  			HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {
  
  		boolean errorView = false;
  
  		// 将异常处理成ModelAndView
  		if (exception != null) {
  			if (exception instanceof ModelAndViewDefiningException) {
  				logger.debug("ModelAndViewDefiningException encountered", exception);
  				mv = ((ModelAndViewDefiningException) exception).getModelAndView();
  			}
  			else {
  				Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
  				// 使用HandlerExceptionResolver处理异常 
  				mv = processHandlerException(request, response, handler, exception);
  				errorView = (mv != null);
  			}
  		}
  
  		// Did the handler return a view to render?
  		if (mv != null && !mv.wasCleared()) {
  
  			// 这里渲染视图
  			render(mv, request, response);
  			if (errorView) {
  				WebUtils.clearErrorRequestAttributes(request);
  			}
  		}
  		else {
  			if (logger.isDebugEnabled()) {
  				logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
  						"': assuming HandlerAdapter completed request handling");
  			}
  		}
  
  		if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
  			// Concurrent handling started during a forward
  			return;
  		}
  
  		// 触发afterCompletion
  		if (mappedHandler != null) {
  			mappedHandler.triggerAfterCompletion(request, response, null);
  		}
  	}
  ```
  
  DispatcherServlet#processHandlerException()
  ```
  /**
  	 * Determine an error ModelAndView via the registered HandlerExceptionResolvers.
  	 * @param request current HTTP request
  	 * @param response current HTTP response
  	 * @param handler the executed handler, or {@code null} if none chosen at the time of the exception
  	 * (for example, if multipart resolution failed)
  	 * @param ex the exception that got thrown during handler execution
  	 * @return a corresponding ModelAndView to forward to
  	 * @throws Exception if no error ModelAndView found
  	 */
  	protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
  			Object handler, Exception ex) throws Exception {
  
  		// Check registered HandlerExceptionResolvers...
  		ModelAndView exMv = null;
  		for (HandlerExceptionResolver handlerExceptionResolver : this.handlerExceptionResolvers) {
  			exMv = handlerExceptionResolver.resolveException(request, response, handler, ex);
  			if (exMv != null) {
  				break;
  			}
  		}
  		if (exMv != null) {
  			if (exMv.isEmpty()) {
  				request.setAttribute(EXCEPTION_ATTRIBUTE, ex);
  				return null;
  			}
  			// We might still need view name translation for a plain error model...
  			if (!exMv.hasView()) {
  				exMv.setViewName(getDefaultViewName(request));
  			}
  			if (logger.isDebugEnabled()) {
  				logger.debug("Handler execution resulted in exception - forwarding to resolved error view: " + exMv, ex);
  			}
  			WebUtils.exposeErrorRequestAttributes(request, ex, getServletName());
  			return exMv;
  		}
  
  		throw ex;
  	}
  ```
  
  
  DispatcherServlet#render()  
  ```
  /**
  	 * Render the given ModelAndView.
  	 * <p>This is the last stage in handling a request. It may involve resolving the view by name.
  	 * @param mv the ModelAndView to render
  	 * @param request current HTTP servlet request
  	 * @param response current HTTP servlet response
  	 * @throws ServletException if view is missing or cannot be resolved
  	 * @throws Exception if there's a problem rendering the view
  	 */
  	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
  		// Determine locale for request and apply it to the response.
  		Locale locale = this.localeResolver.resolveLocale(request);
  		response.setLocale(locale);
  
  
  		// 解析视图 View
  		View view;
  		if (mv.isReference()) {
            // 使用viewRsolver根据视图名字解析视图View
  			// We need to resolve the view name.
  			view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
  			if (view == null) {
  				throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
  						"' in servlet with name '" + getServletName() + "'");
  			}
  		}
  		else {
  			// No need to lookup: the ModelAndView object contains the actual View object.
  			view = mv.getView();
  			if (view == null) {
  				throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
  						"View object in servlet with name '" + getServletName() + "'");
  			}
  		}
  
  		// 委派View对象去呈现视图到response响应
  		// Delegate to the View object for rendering.
  		if (logger.isDebugEnabled()) {
  			logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
  		}
  		try {
  			if (mv.getStatus() != null) {
  				response.setStatus(mv.getStatus().value());
  			}
  			view.render(mv.getModelInternal(), request, response);
  		}
  		catch (Exception ex) {
  			if (logger.isDebugEnabled()) {
  				logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
  						getServletName() + "'", ex);
  			}
  			throw ex;
  		}
  	}
  ```
  
  DispatcherServlet#resolveViewName()
  ```
  /**
  	 * Resolve the given view name into a View object (to be rendered).
  	 * <p>The default implementations asks all ViewResolvers of this dispatcher.
  	 * Can be overridden for custom resolution strategies, potentially based on
  	 * specific model attributes or request parameters.
  	 * @param viewName the name of the view to resolve
  	 * @param model the model to be passed to the view
  	 * @param locale the current locale
  	 * @param request current HTTP servlet request
  	 * @return the View object, or {@code null} if none found
  	 * @throws Exception if the view cannot be resolved
  	 * (typically in case of problems creating an actual View object)
  	 * @see ViewResolver#resolveViewName
  	 */
  	protected View resolveViewName(String viewName, Map<String, Object> model, Locale locale,
  			HttpServletRequest request) throws Exception {
       // 循环调用ViewResolver, 根据视图名字,解析成View
  		for (ViewResolver viewResolver : this.viewResolvers) {
  			View view = viewResolver.resolveViewName(viewName, locale);
  			if (view != null) {
  				return view;
  			}
  		}
  		return null;
  	}
  ```