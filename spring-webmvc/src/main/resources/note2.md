



#### dispatcherServlet各组件的初始化及调用过程

##### 1. handleMapping
  
  
  1.1) handlerMappings 属性
  
  ```
  /** List of HandlerMappings used by this servlet */
  	private List<HandlerMapping> handlerMappings;
  ```

  
  1.2) 初始化  
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

  1.3) doDispatch()中获取handlerMapping
  
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




