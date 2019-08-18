



#### dispatcherServlet各组件的初始化及调用过程

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
   
   
   
##### 3. xxx

   ###### 3.1) xxx属性 
   ###### 3.2) 初始化 
   ###### 3.3) doDispatch()中获取xxx



##### 3. xxx

   ###### 3.1) xxx属性 
   ###### 3.2) 初始化 
   ###### 3.3) doDispatch()中获取xxx  
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
   			View view = viewResolver.resolveViewName(viewName, locale);
   			if (view != null) {
   				return view;
   			}
   		}
   		return null;
   	}
   ```
   

##### 3. xxx

   ###### 3.1) xxx属性 
   ###### 3.2) 初始化 
   ###### 3.3) doDispatch()中获取xxx
   
   
   
   
   
   
   
   
      
   