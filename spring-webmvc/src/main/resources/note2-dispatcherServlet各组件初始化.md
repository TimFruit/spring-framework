



#### dispatcherServlet各组件的初始化及调用过程

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

以下主要组件的初始化及调用过程

##### 1. handleMapping 
  
  
  ###### 1.1) handlerMappings 属性
  
  ```
  /** List of HandlerMappings used by this servlet */
  	private List<HandlerMapping> handlerMappings;
  ```

  
  ###### 1.2) 初始化  
  dispatcherServlet#initHandlerMappings(applicationContext context)
  ```
  /**
  	 * Initialize the HandlerMappings used by this class.
  	 * <p>If no HandlerMapping beans are defined in the BeanFactory for this namespace,
  	 * we default to BeanNameUrlHandlerMapping.
  	 */
  	private void initHandlerMappings(ApplicationContext context) {
  		this.handlerMappings = null;
  
  		if (this.detectAllHandlerMappings) {
  		    // 1. 使用探测功能, 从上下文中查找所有的handlerMappings, 包括父上下文  
  			// Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
  			Map<String, HandlerMapping> matchingBeans =
  					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
  			if (!matchingBeans.isEmpty()) {
  				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
  				// We keep HandlerMappings in sorted order.
  				AnnotationAwareOrderComparator.sort(this.handlerMappings);
  			}
  		}
  		else {
  		    // 2. 从上下文中获取bean 
  			try {
  				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
  				this.handlerMappings = Collections.singletonList(hm);
  			}
  			catch (NoSuchBeanDefinitionException ex) {
  				// Ignore, we'll add a default HandlerMapping later.
  			}
  		}
  
        // 使用默认的handlerMappings
  		// Ensure we have at least one HandlerMapping, by registering
  		// a default HandlerMapping if no other mappings are found.
  		if (this.handlerMappings == null) {
  			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
  			if (logger.isDebugEnabled()) {
  				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
  			}
  		}
  	}
  ```

  ###### 1.3) doDispatch()中获取handlerMapping
  
  ```
  /**
  	 * Return the HandlerExecutionChain for this request.
  	 * <p>Tries all handler mappings in order.
  	 * @param request current HTTP request
  	 * @return the HandlerExecutionChain, or {@code null} if no handler could be found
  	 */
  	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
  		for (HandlerMapping hm : this.handlerMappings) {
  			if (logger.isTraceEnabled()) {
  				logger.trace(
  						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
  			}
  			HandlerExecutionChain handler = hm.getHandler(request);
  			if (handler != null) {
  				return handler;
  			}
  		}
  		return null;
  	}
  ```


##### 2. handlerAdapter

   ###### 2.1) handlerAdapter 属性 
   ```
   /** List of HandlerAdapters used by this servlet */
   	private List<HandlerAdapter> handlerAdapters;
   ```
   
   ###### 2.2) 初始化
   ```
   /**
   	 * Initialize the HandlerAdapters used by this class.
   	 * <p>If no HandlerAdapter beans are defined in the BeanFactory for this namespace,
   	 * we default to SimpleControllerHandlerAdapter.
   	 */
   	private void initHandlerAdapters(ApplicationContext context) {
   		this.handlerAdapters = null;
   
   		if (this.detectAllHandlerAdapters) {
   			// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
   			Map<String, HandlerAdapter> matchingBeans =
   					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
   			if (!matchingBeans.isEmpty()) {
   				this.handlerAdapters = new ArrayList<HandlerAdapter>(matchingBeans.values());
   				// We keep HandlerAdapters in sorted order.
   				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
   			}
   		}
   		else {
   			try {
   				HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
   				this.handlerAdapters = Collections.singletonList(ha);
   			}
   			catch (NoSuchBeanDefinitionException ex) {
   				// Ignore, we'll add a default HandlerAdapter later.
   			}
   		}
   
   		// Ensure we have at least some HandlerAdapters, by registering
   		// default HandlerAdapters if no other adapters are found.
   		if (this.handlerAdapters == null) {
   			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
   			if (logger.isDebugEnabled()) {
   				logger.debug("No HandlerAdapters found in servlet '" + getServletName() + "': using default");
   			}
   		}
   	}
   ```
   
   ###### 2.3) doDispatch()中获取HandlerAdapter
   ```
   /**
   	 * Return the HandlerAdapter for this handler object.
   	 * @param handler the handler object to find an adapter for
   	 * @throws ServletException if no HandlerAdapter can be found for the handler. This is a fatal error.
   	 */
   	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
   	    // 循环遍历handlerAdapter, 判断是否支持handlerAdapter 
   		for (HandlerAdapter ha : this.handlerAdapters) {
   			if (logger.isTraceEnabled()) {
   				logger.trace("Testing handler adapter [" + ha + "]");
   			}
   			if (ha.supports(handler)) {
   				return ha;
   			}
   		}
   		throw new ServletException("No adapter for handler [" + handler +
   				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
   	}
   ``` 
   
   
   
##### 3. HandlerExceptionResolver

   ###### 3.1) HandlerExceptionResolver属性 
   ```
   /** List of HandlerExceptionResolvers used by this servlet */
   	private List<HandlerExceptionResolver> handlerExceptionResolvers;
   ```
   ###### 3.2) 初始化 
   ```
   /**
   	 * Initialize the HandlerExceptionResolver used by this class.
   	 * <p>If no bean is defined with the given name in the BeanFactory for this namespace,
   	 * we default to no exception resolver.
   	 */
   	private void initHandlerExceptionResolvers(ApplicationContext context) {
   		this.handlerExceptionResolvers = null;
   
   		if (this.detectAllHandlerExceptionResolvers) {
   		    // 从上下文中查找 
   			// Find all HandlerExceptionResolvers in the ApplicationContext, including ancestor contexts.
   			Map<String, HandlerExceptionResolver> matchingBeans = BeanFactoryUtils
   					.beansOfTypeIncludingAncestors(context, HandlerExceptionResolver.class, true, false);
   			if (!matchingBeans.isEmpty()) {
   				this.handlerExceptionResolvers = new ArrayList<HandlerExceptionResolver>(matchingBeans.values());
   				// We keep HandlerExceptionResolvers in sorted order.
   				AnnotationAwareOrderComparator.sort(this.handlerExceptionResolvers);
   			}
   		}
   		else {
   			try {
   			    // 根据beanName获取
   				HandlerExceptionResolver her =
   						context.getBean(HANDLER_EXCEPTION_RESOLVER_BEAN_NAME, HandlerExceptionResolver.class);
   				this.handlerExceptionResolvers = Collections.singletonList(her);
   			}
   			catch (NoSuchBeanDefinitionException ex) {
   				// Ignore, no HandlerExceptionResolver is fine too.
   			}
   		}
   
        // 默认 
   		// Ensure we have at least some HandlerExceptionResolvers, by registering
   		// default HandlerExceptionResolvers if no other resolvers are found.
   		if (this.handlerExceptionResolvers == null) {
   			this.handlerExceptionResolvers = getDefaultStrategies(context, HandlerExceptionResolver.class);
   			if (logger.isDebugEnabled()) {
   				logger.debug("No HandlerExceptionResolvers found in servlet '" + getServletName() + "': using default");
   			}
   		}
   	}
   ```
   ###### 3.3) processHandlerException()中使用HandlerExceptionResolver处理异常
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



##### 4. RequestToViewNameTranslator

   ###### 4.1) RequestToViewNameTranslator属性 
   ```
   /** RequestToViewNameTranslator used by this servlet */
   	private RequestToViewNameTranslator viewNameTranslator;
   ```
   ###### 4.2) 初始化 
   ```
   /**
   	 * Initialize the RequestToViewNameTranslator used by this servlet instance.
   	 * <p>If no implementation is configured then we default to DefaultRequestToViewNameTranslator.
   	 */
   	private void initRequestToViewNameTranslator(ApplicationContext context) {
   		try {
   		    // 从上下文中根据beanName获取
   			this.viewNameTranslator =
   					context.getBean(REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME, RequestToViewNameTranslator.class);
   			if (logger.isDebugEnabled()) {
   				logger.debug("Using RequestToViewNameTranslator [" + this.viewNameTranslator + "]");
   			}
   		}
   		catch (NoSuchBeanDefinitionException ex) {
   			// We need to use the default.
   			this.viewNameTranslator = getDefaultStrategy(context, RequestToViewNameTranslator.class);
   			if (logger.isDebugEnabled()) {
   				logger.debug("Unable to locate RequestToViewNameTranslator with name '" +
   						REQUEST_TO_VIEW_NAME_TRANSLATOR_BEAN_NAME + "': using default [" + this.viewNameTranslator +
   						"]");
   			}
   		}
   	}
   ```
   ###### 4.3) doDispatch()中用于请求转换成默认的viewName
   ```
   /**
   	 * Do we need view name translation?
   	 */
   	private void applyDefaultViewName(HttpServletRequest request, ModelAndView mv) throws Exception {
   		if (mv != null && !mv.hasView()) {
   			mv.setViewName(getDefaultViewName(request));
   		}
   	}
   ```
   ```
   /**
   	 * Translate the supplied request into a default view name.
   	 * @param request current HTTP servlet request
   	 * @return the view name (or {@code null} if no default found)
   	 * @throws Exception if view name translation failed
   	 */
   	protected String getDefaultViewName(HttpServletRequest request) throws Exception {
   		return this.viewNameTranslator.getViewName(request);
   	}
   ```
   
   

##### 5. ViewResolver

   ###### 5.1) ViewResolver属性 
   ```
      /** List of ViewResolvers used by this servlet */
      private List<ViewResolver> viewResolvers;
  ```
   ###### 5.2) 初始化 
   ```
   /**
   	 * Initialize the ViewResolvers used by this class.
   	 * <p>If no ViewResolver beans are defined in the BeanFactory for this
   	 * namespace, we default to InternalResourceViewResolver.
   	 */
   	private void initViewResolvers(ApplicationContext context) {
   		this.viewResolvers = null;
   
   		if (this.detectAllViewResolvers) {
   			// Find all ViewResolvers in the ApplicationContext, including ancestor contexts.
   			Map<String, ViewResolver> matchingBeans =
   					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, ViewResolver.class, true, false);
   			if (!matchingBeans.isEmpty()) {
   				this.viewResolvers = new ArrayList<ViewResolver>(matchingBeans.values());
   				// We keep ViewResolvers in sorted order.
   				AnnotationAwareOrderComparator.sort(this.viewResolvers);
   			}
   		}
   		else {
   			try {
   				ViewResolver vr = context.getBean(VIEW_RESOLVER_BEAN_NAME, ViewResolver.class);
   				this.viewResolvers = Collections.singletonList(vr);
   			}
   			catch (NoSuchBeanDefinitionException ex) {
   				// Ignore, we'll add a default ViewResolver later.
   			}
   		}
   
   		// Ensure we have at least one ViewResolver, by registering
   		// a default ViewResolver if no other resolvers are found.
   		if (this.viewResolvers == null) {
   			this.viewResolvers = getDefaultStrategies(context, ViewResolver.class);
   			if (logger.isDebugEnabled()) {
   				logger.debug("No ViewResolvers found in servlet '" + getServletName() + "': using default");
   			}
   		}
   	}
   ```
   
   ###### 5.3) resolveViewName()中使用viewResolver解析View
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
  
        for (ViewResolver viewResolver : this.viewResolvers) {
            // 根据名字解析获取view
            View view = viewResolver.resolveViewName(viewName, locale);
            if (view != null) {
                return view;
            }
        }
        return null;
    }
  ```
   
   

   
   
   
   
      
   